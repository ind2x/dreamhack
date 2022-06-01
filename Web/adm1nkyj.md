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
: 이 값은 pw 컬럼명을 담고 있다. 

출력 결과 pw 컬럼명은 ```xPw4coaa1sslfe``` 이다.

<br>


