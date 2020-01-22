# coffee_break

## Task

The program "encrypt.py" gets one string argument and outputs ciphertext.

Example:

$ python encrypt.py "test_text"
gYYpbhlXwuM59PtV1qctnQ==

The following text is ciphertext with "encrypt.py".

FyRyZNBO2MG6ncd3hEkC/yeYKUseI/CxYoZiIeV2fe/Jmtwx+WbWmU1gtMX9m905

Please download "encrypt.py" from the following url.

```python
import sys
from Crypto.Cipher import AES
import base64


def encrypt(key, text):
    s = ''
    for i in range(len(text)):
        s += chr((((ord(text[i]) - 0x20) + (ord(key[i % len(key)]) - 0x20)) % (0x7e - 0x20 + 1)) + 0x20)
    return s


key1 = "SECCON"
key2 = "seccon2019"
text = sys.argv[1]

enc1 = encrypt(key1, text)
cipher = AES.new(key2 + chr(0x00) * (16 - (len(key2) % 16)), AES.MODE_ECB)
p = 16 - (len(enc1) % 16)
enc2 = cipher.encrypt(enc1 + chr(p) * p)
print(base64.b64encode(enc2).decode('ascii'))

```

## Solution

```python
### ...original script... ###

def decrypt(key, cyphertext):
    text = ''
    for i in range(len(cyphertext)):
        # Changed + to -
        text += chr( (  (   (    ord(chr(cyphertext[i])) - 0x20    ) - (    ord(     key[i % len(key)]     ) - 0x20    )   ) % (0x7e - 0x20 + 1)  ) + 0x20)
    return text

cypher = base64.b64decode("FyRyZNBO2MG6ncd3hEkC/yeYKUseI/CxYoZiIeV2fe/Jmtwx+WbWmU1gtMX9m905")
dec2 = cipher.decrypt(cypher)
print(decrypt(key1,dec2))

```