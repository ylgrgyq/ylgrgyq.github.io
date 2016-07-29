---                                                                                                                                                                                                     
layout: post
title:  "追踪 Netty 异常占用堆外内存的经验分享" 
category: JAVA
date:   2016-07-17 08:21:30 +0800
tags: [NETTY, JAVA]
--- 

本文记述了定位 Netty 的一处漏洞的全过程。事情的起因是我们一个使用了 [Netty](http://netty.io) 的服务，随着运行时间的增长，其占用的堆外内存会逐步攀升，大概持续运行三四天左右，堆外内存会被全部占满，然后只能重启来解决问题。好在服务是冗余配置的，并且可以自动进行 Load Balance，所以每次重启不会带来什么损失。

从现象上分析，我们能确定一定是服务内部有地方出现了内存泄露。在这个出问题的服务上有大量的网络 IO 操作，为了优化性能，我们使用了 PooledByteBufAllocator 来分配 PooledDirectByteBuf。因为是堆外内存泄露，所以第一种可能就是我们在某个地方分配了内存但忘记了释放。我们仔细检查了与业务相关的 ChannelHandler 但并未发现问题，于是又将 Netty 的 io.netty.leakDetectionLevel 设置到 Advanced 级别，放在 Beta 环境上进行测试。在服务连续运行了几天并即将因内存不足再次重启之前，我们从日志中也没有发现任何由 Netty 打出来的内存泄露的报警信息。随后我们又将 io.netty.leakDetectionLevel 设置到 Paranoid 来重新测试，但依然没有发现有关 Netty 内存泄露的相关日志。

在排查过程中，我们也发现虽然引起服务重启的原因是堆外内存不足，但实际堆内内存也有小幅度攀升。起初我们以为这是正常现象，因为有使用 PooledByteBufAllocator，这种 Allocator 为了减少堆外内存的重复分配，会在服务内部建立一个堆外内存池，每次分配内存优先从内存池分配，只有内存池没有足够内存时候，才会去堆外分配新内存。内存池上的内存虽然在堆外，但维护内存池的数据结构却是在堆上。随着堆外内存分配的增多，内部维护内存池的数据结构也会相应增大，堆内内存也会有所升高。为了验证这个猜想，我们将 io.netty.allocator.type 设置为 unpooled 再去测试，几天后发现堆内内存依旧会小幅度攀升，从而判定内存泄露并不是由内存池而导致。

## 顺藤摸瓜

不是内存池出现泄露，而且堆内堆外一起泄露，能同时占用堆内堆外内存的对象一般不多，不过一时也想不出到底有哪些，于是随手 dump 了一份堆内存快照开始分析，果不其然从中还真看出了些端倪。一般通过 dump 排查内存泄露都使用 [Eclipse Memory Analyzer Tool](http://www.eclipse.org/mat/)（简称 MAT）去检查 dominator tree，从中找出哪个类的对象不正常地占用了大量内存。但这次的 dominator tree 看不出有什么问题。因为出现泄露的对象在堆上占用的总内存并不是很多，它在 dominator tree 上根本排不到前列，很难被关注到，但是在 Histogram（如下图）中就有它的身影了。

![2016-07-17 7 40 25](https://cloud.githubusercontent.com/assets/1115061/16897985/53d4ae34-4bf8-11e6-8170-acc9eee1a42c.png)

出现泄露的就是上图画红框圈起来的 OpenSslClientContext。

从 OpenSslClientContext 的使用也能看出来，这个出问题的服务是作为 client 一端，使用 OpenSsl 去连接另一个服务。一般正常使用的情况下，一个 SSL 证书会只对应一个 OpenSslClientContext。对于大多数场景来说，整个服务可能只会使用一种证书，所以只会有一个 OpenSslClientContext 保留在内存中。但我们这个服务有些特殊，会使用很多不同的证书去建立 SSL 连接，只是服务在内部做了限制，将同一时刻不同证书建立的 SSL 连接数量控制在几十个左右，并且在一个 SSL 证书使用完毕之后，指向该 SSL 证书对应 OpenSslClientContext 的引用会被清理掉。之后按正常逻辑来说 OpenSslClientContext 会被 GC 掉，不会在内存中长久停留。但是上图显示同一时间并存的 OpenSslClientContext 有 27472 个之多，远远超过了原本服务内部在同一时间允许并存的 OpenSslClientContext 的数量限制，这就意味着这个 OpenSslClientContext 发生了泄露。

从 dump 中我们还发现，维护 OpenSslClientContext 的业务对象没有产生泄露，并被正常 GC。这说明我们的业务代码可以正确清理指向 OpenSslClientContext 对象的引用。那这个 OpenSslClientContext 是怎么被 GC Root 引用到的呢？

## 水落石出

依然是使用 MAT，分析指向泄露的 OpenSslClientContext 对象的引用路径后得到如下图：

![2016-07-13 7 27 37](https://cloud.githubusercontent.com/assets/1115061/16897983/472ae7fc-4bf8-11e6-8a69-a311827348b0.png)

可以看出 OpenSslClientContext 对象指向了两个引用，一个是 Finalizer 上的引用，一个是 Native Stack 上的引用，这表明我们的业务对象已经正确地释放了对 OpenSslClientContext 的引用。
Finalizer 引用的存在是因为 finalize method 被 OpenSslClientContext  所 overide 了（实际是 OpenSslClientContext 的父类 OpenSslContext 来进行 overide），这样 JVM 会为这类对象自动加上 Finalizer 引用，从而在该对象被 GC 的时候调用对象的 finalize method。但这个 Finalizer 引用不会阻碍对象被 GC，所以内存泄露与它没有直接的关系。

而 Native Stack 就是 GC Root，被其引用的对象是不能被 GC 的，这也就是 OpenSslClientContext 泄露的源头。从这个 Native Stack 指向的对象的类名 OpenSslClientContext$1 能看出，这是一个 OpenSslClientContext 上的匿名类。

查看这个匿名类的对象的属性：

![2016-07-13 9 57 15](https://cloud.githubusercontent.com/assets/1115061/16897981/426fd038-4bf8-11e6-95d7-6b657213c973.png)

一方面它包含有指向外部 OpenSslClientContext 的引用 this$0，还包含一个叫做 val$extendedManager 的引用指向了对象 sun.security.ssl.X509TrustManagerImpl。这时候去翻看 Netty 4.1.1-Final 的 OpenSslClientContext 第 240 ~ 268 行代码如下（注意现在的 Netty 4.1 分支已经将这个 bug 修复，所以不能直接看到下面的代码了）：

{% highlight java linenos %}
try {
    if (trustCertCollection != null) {
        trustManagerFactory = buildTrustManagerFactory(trustCertCollection, trustManagerFactory);
    } else if (trustManagerFactory == null) {
        trustManagerFactory = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init((KeyStore) null);
    }
    final X509TrustManager manager = chooseTrustManager(trustManagerFactory.getTrustManagers());

    // Use this to prevent an error when running on java < 7
    if (useExtendedTrustManager(manager)) {
        final X509ExtendedTrustManager extendedManager = (X509ExtendedTrustManager) manager;
        SSLContext.setCertVerifyCallback(ctx, new AbstractCertificateVerifier() {
            @Override
            void verify(OpenSslEngine engine, X509Certificate[] peerCerts, String auth)
                    throws Exception {
                extendedManager.checkServerTrusted(peerCerts, auth, engine);
            }
        });
    } else {
        SSLContext.setCertVerifyCallback(ctx, new AbstractCertificateVerifier() {
            @Override
            void verify(OpenSslEngine engine, X509Certificate[] peerCerts, String auth)
                    throws Exception {
                manager.checkServerTrusted(peerCerts, auth);
            }
        });
    }
} catch (Exception e) {
    throw new SSLException("unable to setup trustmanager", e);
}
{% endhighlight %}

对比上面 MAT 中看到的 val$extendedManager 引用信息我们会知道，上述代码 14 ~ 20 行设置的这个 callback 就是之前说的出现泄露的匿名类。匿名类有指向外部对象 OpenSslClientContext 的引用，也有个指向外部 extendedManager 的引用。这段逻辑是在 OpenSslClientContext 的构造函数中的，而且 12 ~ 29 行的这个 if 语句无论走哪个分支，都会设置一个匿名的 verifier 到 SSLContext.setCertVerifyCallback，也就是说只要 new 一个 OpenSslClientContext 对象，就一定会设置一个 verifier 到 Native Stack 上。

找到 SSLContext.setCertVerifyCallback 的代码。在我们使用的 netty-tcnative-1.1.33.Fork17 中，SSLContext.setCertVerifyCallback 函数声明如下：

{% highlight java %}
/**
 * Allow to hook {@link CertificateVerifier} into the handshake processing.
 * This will call {@code SSL_CTX_set_cert_verify_callback} and so replace the default verification
 * callback used by openssl
 * @param ctx Server or Client context to use.
 * @param verifier the verifier to call during handshake.
 */
public static native void setCertVerifyCallback(long ctx, CertificateVerifier verifier);
{% endhighlight %}

从注释上能看出来，这个函数是用来让用户自定义证书检查函数，好在 SSL Handshake 过程中来使用去校验证书。

函数声明上的「native」关键字也表明它是通过调用本地 C 代码实现的。结合之前的分析，能推理出一定是这个 Native 代码将 verifier callback 存入了 Native Stack，并且在 OpenSslClientContext 没有其他引用指向时没能将这个 callback 正确清理，从而让 OpenSslClientContext 对象有了从 GC Root 过来的引用指向，所以不能被 GC 掉，造成了泄露。

有了指导路线，我们继续追踪问题。在 netty-tcnative-1.1.33.Fork17 的 sslcontext.c 文件下找到 setCertVerifyCallback 函数对应的 Native 代码如下：

{% highlight c linenos %}
TCN_IMPLEMENT_CALL(void, SSLContext, setCertVerifyCallback)(TCN_STDARGS, jlong ctx, jobject verifier)
{
    tcn_ssl_ctxt_t *c = J2P(ctx, tcn_ssl_ctxt_t *);

    UNREFERENCED(o);
    TCN_ASSERT(ctx != 0);

    if (verifier == NULL) {
        SSL_CTX_set_cert_verify_callback(c->ctx, NULL, NULL);
    } else {
        jclass verifier_class = (*e)->GetObjectClass(e, verifier);
        jmethodID method = (*e)->GetMethodID(e, verifier_class, "verify", "(J[[BLjava/lang/String;)I");

        if (method == NULL) {
            return;
        }    
        // Delete the reference to the previous specified verifier if needed.
        if (c->verifier != NULL) {
            (*e)->DeleteLocalRef(e, c->verifier);
        }    
        c->verifier = (*e)->NewGlobalRef(e, verifier);
        c->verifier_method = method;

        SSL_CTX_set_cert_verify_callback(c->ctx, SSL_cert_verify, NULL);
    }    
}       
{% endhighlight %}

这里函数的 verifier 参数就对应着 SSLContext.setCertVerifyCallback 上传入的 verifier。这里也不需要完全理解上面代码的含义，主要是看到第 21 行，创建了个引用从 \*e 指向了 verifier。这个 \*e 是个 JNIEnv struct，NewGlobalRef(e, verifier) 相当于将 verifier 保存在一个全局的变量当中，必须通过对应的 DeleteGlobalRef 才能销毁。

在搜索 sslcontext.c 的代码后发现在正常的逻辑下，要对 verifier 调用 DeleteGlobalRef 将其清理，必须调用 SSLContext.free 函数才能实现。SSLContext.free 声明如下：

{% highlight java %}
/**
 * Free the resources used by the Context
 * @param ctx Server or Client context to free.
 * @return APR Status code.
 */
public static native int free(long ctx);
{% endhighlight %}

它还有个对应的 make 函数，合并起来用于负责 OpenSslClientContext 分配和回收一些 Native 的资源。OpenSslClientContext 在构造函数中必须调用一次 SSLContext.make，在对象被销毁时需要调用 SSLContext.free。「在对象被销毁时调用」听上去有点析构函数的意思，但 Java 中没有析构函数的概念，看上去 Netty 也没有好的方法来实现这种类似析构函数的功能，虽然所有讲到 finalize 的地方都在谆谆告诫开发者只是知晓它的存在就好但永远不要去使用，Netty 还是「被逼无奈」地将用于资源回收的 SSLContext.free 调用放在了 OpenSslClientContext 的 finalize（继承自 OpenSslContext）函数中。

分析到这里基本就能得到 OpenSslClientContext 泄露的原因了。因为 OpenSslClientContext 在构造时会将一个匿名的 AbstractCertificateVerifier 子类对象作为证书的校验函数（简称为 verifier），通过调用 SSLContext.setCertVerifyCallback 存储到 Native Stack 上，必须在 OpenSslClientContext 销毁时主动调用 SSLContext.free 才能将这个 verifier 从 Native Stack 清除。而 SSLContext.free 是在 OpenSslClientContext 的 finalize 内，必须等到 OpenSslClientContext 被 GC 掉之后才会被调用。由于 verifier 是个匿名类，它含有隐含的指向了其所属 OpenSslClientContext 的引用，导致当 verifier 不被销毁时，其所在 OpenSslClientContext 也无法销毁，从而产生依赖环，verifier 的清理依赖 OpenSslClientContext 的清理，OpenSslClientContext 的清理又依赖 verifier 的清理。这种依赖环如果都是在堆内，JVM GC 的时候会自动检测依赖环，并将相互依赖的两个对象全部 GC 掉。但这里 verifier 比较特殊，它是直接存储在 Native Stack 上的，JVM GC 拿它没有办法。JVM GC 的管辖范围只有堆，Native Stack 可以理解为是它的上级，它无权过问。

另外补充一点，上述问题虽然是在 OpenSslClientContext 中发现，但 OpenSslServerContext 中也有相同问题。

# 修复

Bug 找到了，具体的 PR 请参考[这里](https://github.com/netty/netty/pull/5380)。修复办法就是将导致泄露的匿名 AbstractCertificateVerifier 子类对象修改为 static 的内部类，这样它不会包含指向其所在外部类的引用（即 OpenSslClientContext），从而不会阻碍外部类的 GC，也就避免了泄露的发生。

问题是解决了，但究其根本原因是不是可以归结到 finalize 函数的使用呢？如果 OpenSslClientContext 没有使用 finalize，而是暴露一个类似 close 的接口，要求 OpenSslClientContext 的使用者主动调用 close，finalize 内只是打印日志，提醒使用者没有调用 close，这个问题是不是从一开始就不会存在了呢？
一般来说 finalize 出现的问题主要有以下几类：

1. 使用 finalize 的对象在创建和销毁时性能比正常的对象差；
2. finalize 执行时间不确定。可能出现 heap 内虽然有很多拥有 finalize 函数的类对象，且这些对象都已死掉（从 GC Root 无法访问），如果遇到 GC 压力比较大等原因，这些对象的 finalize 还没有被触发，就会导致这些本来该被 GC 但没有被 GC 的对象大量存在于 Heap 中。

猜想 Netty 这里使用 finalize 而不是明确提供一个 close 函数，主要是为了使用方便，毕竟 OpenSslContext 在大多数场景下在一个服务中只存在一两个对象，需要将其销毁的情况也许也不是很多。以上猜想[在这个 issue ](https://github.com/netty/netty/issues/4958)中也得到了一定程度的印证，[并且 Netty 已经在修改这个问题](https://github.com/netty/netty/pull/5547)，让 OpenSslContext 实现 ReferenceCount 接口，在 finalize 之外又提供了 release 函数专门用于清理 Native 资源。

所以分享这些经验来让大家引以为戒，finalize 要尽量少用，看着以为使用 finalize 很合理的地方还是有可能出现问题。
