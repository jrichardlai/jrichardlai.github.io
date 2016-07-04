---
title: "Enable TLSv1.1 for Android with React Native"
description: for PCI compliance
---

At [TaskRabbit](https://www.taskrabbit.com) for security purposes and for PCI compliance (which is a set of security standards to follow when processing Credit Cards) we needed to enforce TLSv1.1+.

Unfortunately in Android, until API 20 TLSv1.1 is not enabled by default as though it is available from API 16 😞.

### How to set up the Android app

In React Native 0.27, `OkHttp` was upgraded to V3. It’s great, this means multiple are fixed for HTTP/2 and as `OkHttp2` features are frozen.

With previous versions of React Native you could get the client with `OkHttpClientProvider.getOkHttpClient();` then set the SSL Socket Factory and connection Specs, however with `OKHttp3` the client is now immutable. Here are some notes on how to enable TLSv1.1 for your app (you can change to TLSv1.2 with the same purpose).

You will need to use a SSL Socket Factory, here is an implementation inspired by [this gist](https://gist.github.com/mlc/549409f649251897ebef).

```
import java.io.IOException;
import java.net.InetAddress;
import java.net.Socket;
import java.net.UnknownHostException;

import javax.net.ssl.SSLSocket;
import javax.net.ssl.SSLSocketFactory;

/**
 * Taken from https://gist.github.com/mlc/549409f649251897ebef
 *
 * Enables TLS when creating SSLSockets.
 *
 * @link https://developer.android.com/reference/javax/net/ssl/SSLSocket.html
 * @see SSLSocketFactory
 */
class TLSSocketFactory extends SSLSocketFactory {
    final SSLSocketFactory delegate;

    public TLSSocketFactory(SSLSocketFactory delegate) {
        this.delegate = delegate;
    }

    @Override
    public String[] getDefaultCipherSuites() {
        return delegate.getDefaultCipherSuites();
    }

    @Override
    public String[] getSupportedCipherSuites() {
        return delegate.getSupportedCipherSuites();
    }

    @Override
    public Socket createSocket(Socket s, String host, int port, boolean autoClose) throws IOException {
        return patch(delegate.createSocket(s, host, port, autoClose));
    }

    @Override
    public Socket createSocket(String host, int port) throws IOException, UnknownHostException {
        return patch(delegate.createSocket(host, port));
    }

    @Override
    public Socket createSocket(String host, int port, InetAddress localHost, int localPort) throws IOException, UnknownHostException {
        return patch(delegate.createSocket(host, port, localHost, localPort));
    }

    @Override
    public Socket createSocket(InetAddress host, int port) throws IOException {
        return patch(delegate.createSocket(host, port));
    }

    @Override
    public Socket createSocket(InetAddress address, int port, InetAddress localAddress, int localPort) throws IOException {
        return patch(delegate.createSocket(address, port, localAddress, localPort));
    }

    private Socket patch(Socket s) {
        if (s instanceof SSLSocket) {
            ((SSLSocket) s).setEnabledProtocols(((SSLSocket) s).getSupportedProtocols());
        }
        return s;
    }
}
```

Create a `TLSSetup` class:

```
import android.util.Log;
import com.facebook.react.modules.network.OkHttpClientProvider;
import com.facebook.react.modules.network.ReactCookieJarContainer;
import okhttp3.ConnectionSpec;
import okhttp3.OkHttpClient;
import okhttp3.TlsVersion;

import javax.net.ssl.*;
import java.util.Arrays;
import java.util.concurrent.TimeUnit;

public class TLSSetup {

    static String TAG = "TLSSetup";

    public static void configure(){
        try {
            SSLContext sc = SSLContext.getInstance("TLSv1.1");
            sc.init(null, null, null);
            ConnectionSpec cs = new ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
                    .tlsVersions(TlsVersion.TLS_1_2, TlsVersion.TLS_1_1)
                    .build();
            // Taken from OkHttpClientProvider.java
            // Set no timeout by default
            OkHttpClient sClient = new OkHttpClient.Builder()
                    .connectTimeout(0, TimeUnit.MILLISECONDS)
                    .readTimeout(0, TimeUnit.MILLISECONDS)
                    .writeTimeout(0, TimeUnit.MILLISECONDS)
                    .cookieJar(new ReactCookieJarContainer())
                    // set sslSocketFactory
                    .sslSocketFactory(new TLSSocketFactory(sc.getSocketFactory()))
                    // set connectionSpecs
                    .connectionSpecs(Arrays.asList(cs, ConnectionSpec.COMPATIBLE_TLS, ConnectionSpec.CLEARTEXT))
                    .build();

            OkHttpClientProvider.replaceOkHttpClient(sClient);
        } catch (Exception e) {
            Log.e(TAG, e.getMessage());
        }
    }

}
```

In your MainActivity.java call `TLSSetup.configure();`

### Prepare to the TLS change

Now that you have an app that supports TLSv1.1, if the app has yet to be released you are good to go 😊,
however if you currently have users of your app, it is a little more complex as you have to migrate your users to the new version of the app.

Here are some steps you can take to make the migration fluid.

**Notes**: Those points are depending on how much time you have before disabling TLSv1.

1. Release a build with the TLS change.
2. If you can, add an inside app notification about the upgrade of the app.
3. Let users update organically for few days.
4. Notify users about the requirements to upgrade the app by email and pushes (target if possible users that you detect will face the issue, this could be done by collecting the users which TLS used is TLSv1)
5. Let users update organically for few days.
6. If you can, enable force upgrade of the App.
7. 1 or 2 weeks before the TLS change repeat step 4 to 5.
8. Disable TLSv1.

Et voilà.
