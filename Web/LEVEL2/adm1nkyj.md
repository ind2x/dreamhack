## [wargame.kr] adm1nkyj (풀이 봄)
---

```php
<?php
    error_reporting(0);
    
    include("./config.php"); // hidden column name, $FLAG.

    mysql_connect("localhost","adm1nkyj","adm1nkyj_pz");
    mysql_select_db("adm1nkyj");

    /**********************************************************************************************************************/

    function rand_string()
    {
        $string = "ABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890abcdefghijklmnopqrstuvwxyz";
        return str_shuffle($string);
    }

    function reset_flag($count_column, $flag_column)
    {
        $flag = rand_string();
        $query = mysql_fetch_array(mysql_query("SELECT $count_column, $flag_column FROM findflag_2"));
        if($query[$count_column] == 150)
        {
            if(mysql_query("UPDATE findflag_2 SET $flag_column='{$flag}';"))
            {
                mysql_query("UPDATE findflag_2 SET $count_column=0;");
                echo "reset flag<hr>";
            }
            return $flag;
        }
        else
        {
            mysql_query("UPDATE findflag_2 SET $count_column=($query[$count_column] + 1);");
        }
        return $query[$flag_column];
    }

    function get_pw($pw_column){
        $query = mysql_fetch_array(mysql_query("select $pw_column from findflag_2 limit 1"));
        return $query[$pw_column];
    }

    /**********************************************************************************************************************/

    $tmp_flag = "";
    $tmp_pw = "";
    $id = $_GET['id'];
    $pw = $_GET['pw'];
    $flags = $_GET['flag'];
    if(isset($id))
    {
        if(preg_match("/information|schema|user/i", $id) || substr_count($id,"(") > 1) exit("no hack");
        if(preg_match("/information|schema|user/i", $pw) || substr_count($pw,"(") > 1) exit("no hack");
        $tmp_flag = reset_flag($count_column, $flag_column);
        $tmp_pw = get_pw($pw_column);
        $query = mysql_fetch_array(mysql_query("SELECT * FROM findflag_2 WHERE $id_column='{$id}' and $pw_column='{$pw}';"));
        if($query[$id_column])
        {
            if(isset($pw) && isset($flags) && $pw === $tmp_pw && $flags === $tmp_flag)
            {
                echo "good job!!<br />FLAG : <b>".$FLAG."</b><hr>";
            }
            else
            {
                echo "Hello ".$query[$id_column]."<hr>";
            }
        }
    } else {
        highlight_file(__FILE__);
    }
?>
```

## Solution
---

핵심은 pw, flag 컬럼명을 알아내는 것이다. 

처음엔 sys schema로 시도했으나 version을 확인해보니 mariadb 5.5.64 버전이다. 

이 버전에는 sys schema가 없는 듯하다. (값이 출력되지 않는다.)

그래서 mysql 스키마로도 해봤으나 어떤 테이블에서도 값이 출력되지 않았다..

해서 풀이를 조금 보았다..

<br>

pw 컬럼명을 알아내는 건 굉장히 간단했다. **(이걸 왜 생각 못했는지 반성해라..)**

우선 나는 union을 통해 ```union select 1,2,3,4,5```을 해서 컬럼이 5개이며 2번째 값이 id 컬럼임을 확인했다.

<br>

```SELECT * FROM findflag_2 WHERE $id_column='{$id}' and $pw_column='{$pw}'```

위의 쿼리에서 id, pw 컬럼이 php 변수로 되어 있어 알 수가 없다. 

하지만 만약 쿼리가 ```SELECT * FROM findflag_2 WHERE $id_column='' union select 1,' and $pw_column=',3,4,5 #'```이 된다면 ```and $pw_column=```이 출력이 될 것이다.

(이 값은 pw 컬럼명을 담고 있다.)

출력 결과 pw 컬럼명은 ```xPw4coaa1sslfe``` 이다.

그래서 비밀번호를 출력해보면 ```!@SA#$!```이다. 이제 flag 값만 구하면 된다.

<br>


대망의 플래그 컬럼과 값을 알아내는 방법이다..

우선 rubiya 테이블은 아래와 같다.

<br>

![image](https://user-images.githubusercontent.com/52172169/171363354-e9400109-48fe-47c4-a0db-8a7c521c90b4.png)

<br>

알아둘 점은 union을 통해 하나의 테이블로 출력하고자 할 때, union 뒤쪽에 있는 쿼리의 결과 값은 앞의 쿼리의 결과 값 밑에 붙여저서 출력된다.

flag 컬럼 값을 어떻게 출력하는가 하면 서브쿼리와 alias를 통해 가져올 수 있다.

<br>

예를 들어, rubiya 테이블에 있는 결과 값을 컬럼명을 다르게 해서 가져오고 싶다라고 할 때 아래와 같은 쿼리로 가져올 수 있다.

```select 1,2,3 union SELECT * FROM rubiya```

<br>

![image](https://user-images.githubusercontent.com/52172169/171366590-3f9e47f3-6f77-4747-a0c8-d7b450edf3b0.png)

<br>

따라서 이 쿼리 결과를 테이블로 사용한다면? 결과는 아래와 같다.

```select * from rubiya where id='' union select * from (select 1,2,3 union select * from rubiya) x```

<br>

![image](https://user-images.githubusercontent.com/52172169/171368628-a4c2b5ca-3e47-4605-9f21-2c4eb8b33ef3.png)

<br>

문제에서는 id 컬럼만 보여주는데, id 컬럼은 2번째에 위치해있다. 

따라서 alias를 통해 플래그가 들어있는 위치를 alias를 해준 다음 그 값을 id 컬럼이 위치한 2번째에 넣어주면 된다.

말로는 모르겠으니 직접 보면 먼저 ```1.rubiya 테이블의 값들을 가져올 건데 컬럼명을 1,flag,3으로 가져온 다음 그 결과를 테이블로 사용```하였다.

두 번째로 그 테이블에서 1,flag,3 컬럼명을 가져왔다.

<br>

```select * from rubiya where id='' union select 1,flag,3 from (select 1,2 as flag,3 union select * from rubiya) x```

<br>

![image](https://user-images.githubusercontent.com/52172169/171370059-7fd4a72d-e803-474a-809e-baa12a9e1c84.png)

<br>

이를 이용해서 findflag_2의 값을 가져올 건데 flag가 있을만한 위치를 alias를 해준 다음 그 결과 값을 테이블로 사용해서 2번째 위치에 alias를 넣어서 출력해준다.

```SELECT * FROM findflag_2 where id='' union select 1,flag,3,4,5 from (select 1,2,3,4 flag,5 union select * from findflag_2) x limit 1,1 #```

<br>

그러면 flag 값이 나온다.

이제 id, pw, flag 모두 구했으므로 입력을 해주면 Flag를 구할 수 있다...

<br><br>

## 풀이 출처
---

Link : <a href="https://posix.tistory.com/27" target="_blank">posix.tistory.com/27</a>
