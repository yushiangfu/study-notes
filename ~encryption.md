```
openssl rsa -pubin -noout -text -inform PEM -in my.pem

# ras:    rsa key processing
# -pubin: specify it's a public key or otherwise, it's treated as a private key
# -noout:   prevent the output of encoded key
# -text:    print out the key in plain representation
```

```
# convert der to c array
xxd -i my.der
```

```
https://stackoverflow.com/questions/2477570/how-to-load-an-rsa-key-from-binary-data-to-an-rsa-structure-using-the-openssl-c
```
