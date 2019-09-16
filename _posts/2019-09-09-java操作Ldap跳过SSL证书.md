---
layout:     post
title:      Ldap 
subtitle:   java连接ldap跳过SSL证书
date:       2019-09-09
author:     JYH
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - Mac
    - Ldap
---
## 配置
macOS版本 10.14.6

## 参考文章
参考  https://blog.csdn.net/weixin_34208283/article/details/91841963  

## JDK版本
参考文章中使用的是JDK_1.8.181 前的版本
我使用的是JDK_1.8.211 于是发生了以下问题

## 代码
import javax.net.ssl.X509TrustManager;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;

public class LTSTrustmanager  implements X509TrustManager {

    @Override
    public void checkClientTrusted(X509Certificate[] arg0, String arg1)
            throws CertificateException {

    }

    @Override
    public void checkServerTrusted(X509Certificate[] arg0, String arg1)
            throws CertificateException {

    }

    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return new java.security.cert.X509Certificate[0];
    }
}


import javax.net.SocketFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManager;
import java.io.IOException;
import java.net.InetAddress;
import java.net.Socket;
import java.net.UnknownHostException;
import java.security.SecureRandom;

public class LTSSSLSocketFactory extends SSLSocketFactory {

    private SSLSocketFactory socketFactory;

    public LTSSSLSocketFactory() {
        try {
            SSLContext ctx = SSLContext.getInstance("TLS");
            ctx.init(null, new TrustManager[]{ new LTSTrustmanager()}, new SecureRandom());
            socketFactory = ctx.getSocketFactory();
        } catch ( Exception ex ) {
            ex.printStackTrace(System.err);
        }
    }

    public static SocketFactory getDefault(){
        return new LTSSSLSocketFactory();
    }

    @Override
    public Socket createSocket(Socket arg0, String arg1, int arg2, boolean arg3) throws IOException {
        return null;
    }

    @Override
    public String[] getDefaultCipherSuites() {
        return socketFactory.getDefaultCipherSuites();
    }

    @Override
    public String[] getSupportedCipherSuites() {
        return socketFactory.getSupportedCipherSuites();
    }

    @Override
    public Socket createSocket(String arg0, int arg1) throws IOException, UnknownHostException {
        return socketFactory.createSocket(arg0, arg1);
    }

    @Override
    public Socket createSocket(InetAddress arg0, int arg1) throws IOException {
        return socketFactory.createSocket(arg0, arg1);
    }

    @Override
    public Socket createSocket(String arg0, int arg1, InetAddress arg2, int arg3) throws IOException, UnknownHostException {
        return socketFactory.createSocket(arg0, arg1, arg2, arg3);
    }

    @Override
    public Socket createSocket(InetAddress arg0, int arg1, InetAddress arg2, int arg3) throws IOException {
        return socketFactory.createSocket(arg0, arg1, arg2, arg3);
    }
}




## 问题
我使用的是JDK_1.8.211，在操作Ldap时发生了异常：
Caused by: java.security.cert.CertificateException: No subject alternative names matching IP address 10.X.X.X found

Caused by: javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names matching IP address 10.X.X.X found

Caused by: javax.naming.CommunicationException: simple bind failed: 10.X.X.X:X [Root exception is javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names matching IP address 10.X.X.X found]

网上查找后发现是SSL证书的问题，但是配置了跳过Ldap的代码，同样的代码在服务器运行没问题，在同事的windows环境运行没问题，于是开始定位问题。

首先，服务器有证书，于是与同事的运行环境做对比，差异存在于JDK版本和操作系统，我的是mac，同事的是Windows。

打断点后发现，在LTSTrustmanager中，同事那边运行了getAcceptedIssuers()的方法，成功返回了X509Certificate[]，产生了证书，而在我这边没有。

两边同步debug比较差异，最初在AbstractTrustManagerWrapper的checkServerTrusted方法中，var3的identificationProtocol存在差异，同事值为null，我这边运行为LDAPS。

跟进运行链，发现在进程启动后，建立连接时发现com.sun.jndi.ldap中，inStream-in-c-identificationProtocol已经被赋值。

网上进行查询，发现在JDK_1.8.181时，为了提高LDAPS(安全LDAP over TLS)连接的稳健性，端点识别默认情况已启用算法，已在LDAP上启用标识。

所以同样的代码运行结果不同。

## 解决方法
定位问题所在后，解决方法就容易的多。

1.降低JDK版本

2.启动时加上 -Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true
将属性设置为true,禁用端点识别算法。






