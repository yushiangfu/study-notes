```
openssl rsa -pubin -noout -text -inform PEM -in my.pem

# ras:    rsa key processing
# -pubin: specify it's a public key or otherwise, it's treated as a private key
# -noout:   prevent the output of encoded key
# -text:    print out the key in plain representation
```
