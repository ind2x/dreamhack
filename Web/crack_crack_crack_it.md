## crack_crack_crack_it
<hr style="border-top: 1px solid;"><br>

Link 
: <a href="https://dreamhack.io/wargame/challenges/330/" target="_blank">dreamhack.io/wargame/challenges/330/</a>

<br>

```
.htaccess crack!

can you local bruteforce attack?
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution (풀이 봄)
<hr style="border-top: 1px solid;"><br>

linux/unix password format 
: <a href="https://www.cyberciti.biz/faq/understanding-etcshadow-file/" target="_blank">cyberciti.biz/faq/understanding-etcshadow-file/</a> 

<br>

문제에서 주어진 값이다.
: ```blueh4g:$1$tKmdMk4o$f2ag61EKYysxJYgI5ckvj0```

<br>

linux, unix password 저장 형식이다.
: ```password format : $id$salt$hashed```

+ ```$id``` : ```$1 -> md5 hash```

+ ```$salt : $tKmdMk4o -> salt```

+ ```$hashed : $f2ag61EKYysxJYgI5ckvj0``` -> hash된 비밀번호, ```salt+password```

<br>

비밀번호가 몇글자인지는 모르지만 ```G4HeulB```로 시작한다고 함. 

따라서 ```salt+password = f2ag61EKYysxJYgI5ckvj0```

일일이 다 brute force 하기에는 힘들어서 풀이를 찾다가.. 그냥 하려했으면 시간낭비 할 뻔 했다는 생각이 들었음.

<br><br>

풀이
: <a href="https://blog.naver.com/is_king/221843606396" target="_blank">blog.naver.com/is_king/221843606396</a>

<br>

passlib(md5_crypt) 
: <a href="https://passlib.readthedocs.io/en/stable/lib/passlib.hash.md5_crypt.html" target="_blank">passlib.readthedocs.io/en/stable/lib/passlib.hash.md5_crypt.html</a>
: <a href="https://ind2x.github.io/posts/ctf_pymodule/#passlibhash" target="_blank">ind2x.github.io/posts/ctf_pymodule/#passlibhash</a>

<br>

itertools 
: <a href="https://docs.python.org/ko/3/library/itertools.html" target="_blank">docs.python.org/ko/3/library/itertools.html</a>
: <a href="https://ind2x.github.io/posts/python_standard_libraries/#itertools" target="_blank">ind2x.github.io/posts/python_standard_libraries/#itertools</a>

<br>

일반적인 md5 해쉬 알고리즘과 Linux/Unix에서 사용하는 md5 알고리즘이 다르다는 것. 
: Linux는 ```md5_crypt(crypt-md5)```

위의 풀이에서 md5 해쉬와 md5_crypt 해쉬의 차이점을 보여줌.

<br>

```python
import string
from itertools import product
from passlib.hash import md5_crypt

text=string.ascii_lowercase+string.digits
password='G4HeulB'
salt='tKmdMk4o'
hash_value='f2ag61EKYysxJYgI5ckvj0'
fail=1

for len in range(1,20) :
    checks=product(text,repeat=len)
    
    for t in checks :
        input=''.join(t)
        trying=password+input
        attack=md5_crypt.using(salt=salt).hash(trying)
        print("trying ... "+trying, " length : ",len)
        crypted_val=attack.split('$')[3]
    
        if crypted_val == hash_value :
            print("success! passwd is ",trying, "hash value is ",crypted_val)
            fail=0
            break
    
    if fail == 0 :
        break
```

<br>

<br><br>
<hr style="border: 2px solid;">
<br><br>
