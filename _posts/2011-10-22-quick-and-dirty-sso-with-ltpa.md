---
layout: post
title: Quick and dirty SSO with LTPA
date: '2011-10-22T19:54:00.001-04:00'
author: Bill Schneider
tags: 
modified_time: '2011-10-22T19:54:46.976-04:00'
blogger_id: tag:blogger.com,1999:blog-9159309.post-836304170743336076
blogger_orig_url: http://wrschneider.blogspot.com/2011/10/quick-and-dirty-sso-with-ltpa.html
---

If you have WebSphere application server in your environment, it is in fact possible to decode the "LtpaToken" cookie in code for quick-and-dirty SSO with non-WebSphere apps.

The main reason you might want to do this is if you have a portal-like application on WebSphere and want to link to other applications on different non-WebSphere servers. This is only useful if WebSphere is your main point of entry.

Here's how you do it:

* Export the LTPA encryption key to a file from WebSphere using the admin console.  You provide a passphrase and a filename.
* Find the `com.ibm.websphere.ltpa.3DESKey` value in the exported file. This is the encrypted key.
* Base64 decode the above key and decrypt with 3DES, using the passphrase provided.  The decrypted value is the actual key for decrypting LTPA tokens.
* Take the `LtpaToken` cookie, base64 decode it, and decrypt it with the key. The legacy LtpaToken cookie (which you can get with "interoperability mode") is encrypted with 3DES; the newer LtpaToken2 cookie uses AES.
* Convert to String and parse.  The string looks like `values%expiration%signature` where the expiration is a standard UNIX timestamp, which you should use to ensure the token is still valid; and the values somewhere will contain the user DN (e.g., uid=user,ou=company,dc=com).  

The Alfresco codebase contains a good example of how to do this in Java: 

http://svn.alfresco.com/repos/alfresco-open-mirror/alfresco/HEAD/root/modules/quickr/source/java/org/alfresco/repo/lotus/ws/impl/auth/LtpaAuthenticator.java

Relevant fragments of code:

{% highlight java %}
    private static final String AES_DECRIPTING_ALGORITHM = "AES/CBC/PKCS5Padding";
    private static final String DES_DECRIPTING_ALGORITHM = "DESede/ECB/PKCS5Padding";

    private byte[] getSecretKey(String ltpa3DESKey, String ltpaPassword) throws Exception
    {
        MessageDigest md = MessageDigest.getInstance("SHA");    
        md.update(ltpaPassword.getBytes());
        byte[] hash3DES = new byte[24];
        System.arraycopy(md.digest(), 0, hash3DES, 0, 20);
        Arrays.fill(hash3DES, 20, 24, (byte) 0);
        final Cipher cipher = Cipher.getInstance(DES_DECRIPTING_ALGORITHM);        
        final KeySpec keySpec = new DESedeKeySpec(hash3DES);
        final Key secretKey = SecretKeyFactory.getInstance("DESede").generateSecret(keySpec);
        cipher.init(Cipher.DECRYPT_MODE, secretKey);
        byte[] secret = cipher.doFinal(Base64.decodeBase64(ltpa3DESKey.getBytes()));
        
        return secret;
    }
    
    private byte[] decrypt(byte[] token, byte[] key, String algorithm) throws Exception
    {
        SecretKey sKey = null;

        if (algorithm.indexOf("AES") != -1)
        {
            sKey = new SecretKeySpec(key, 0, 16, "AES");
        }
        else
        {
            DESedeKeySpec kSpec = new DESedeKeySpec(key);
            SecretKeyFactory kFact = SecretKeyFactory.getInstance("DESede");
            sKey = kFact.generateSecret(kSpec);
        }
        Cipher cipher = Cipher.getInstance(algorithm);

        if (algorithm.indexOf("ECB") == -1)
        {
            if (algorithm.indexOf("AES") != -1)
            {
                IvParameterSpec ivs16 = generateIvParameterSpec(key, 16);
                cipher.init(Cipher.DECRYPT_MODE, sKey, ivs16);
            }
            else
            {
                IvParameterSpec ivs8 = generateIvParameterSpec(key, 8);
                cipher.init(Cipher.DECRYPT_MODE, sKey, ivs8);
            }
        }
        else
        {
            cipher.init(Cipher.DECRYPT_MODE, sKey);
        }
        return cipher.doFinal(token);
    }
    
    private IvParameterSpec generateIvParameterSpec(byte key[], int size)
    {
        byte[] row = new byte[size];
        
        for (int i = 0; i < size; i++)
        {
            row[i] = key[i];
        }
        
        return new IvParameterSpec(row);
    }
// How to use it:
byte[] secretKey = getSecretKey(ltpa3DESKey, ltpaPassword);
byte[] ltpaTokenBytes = Base64.decodeBase64(ltpaToken.getBytes());
String token = new String(decrypt(ltpaTokenBytes, secretKey, DES_DECRIPTING_ALGORITHM));
// or
String token = new String(decrypt(ltpaTokenBytes, secretKey, AES_DECRIPTING_ALGORITHM));

{% endhighlight %}