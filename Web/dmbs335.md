## dbms335
---

```php
<?php 

if (isset($_GET['view-source'])) {
        show_source(__FILE__);
        exit();
}

include("./inc.php"); // Database Connected

function getOperator(&$operator) { 
    switch($operator) { 
        case 'and': 
        case '&&': 
            $operator = 'and'; 
            break; 
        case 'or': 
        case '||': 
            $operator = 'or'; 
            break; 
        default: 
            $operator = 'or'; 
            break; 
}} 

if(preg_match('/session/isUD',$_SERVER['QUERY_STRING'])) {
    exit('not allowed');
}

parse_str($_SERVER['QUERY_STRING']); 
getOperator($operator); 
$keyword = addslashes($keyword);
$where_clause = ''; 

if(!isset($search_cols)) { 
    $search_cols = 'subject|content'; 
} 

$cols = explode('|',$search_cols); 

foreach($cols as $col) { 
    $col = preg_match('/^(subject|content|writer)$/isDU',$col) ? $col : ''; 
    if($col) { 
        $query_parts = $col . " like '%" . $keyword . "%'"; 
    } 

    if($query_parts) { 
        $where_clause .= $query_parts; 
        $where_clause .= ' '; 
        $where_clause .= $operator; 
        $where_clause .= ' '; 
        $query_parts = ''; 
    } 
} 

if(!$where_clause) { 
    $where_clause = "content like '%{$keyword}%'"; 
} 
if(preg_match('/\s'.$operator.'\s$/isDU',$where_clause)) { 
    $len = strlen($where_clause) - (strlen($operator) + 2);
    $where_clause = substr($where_clause, 0, $len); 
} 


?>
<style>
    td:first-child, td:last-child {text-align:center;}
    td {padding:3px; border:1px solid #ddd;}
    thead td {font-weight:bold; text-align:center;}
    tbody tr {cursor:pointer;}
</style>
<br />
<table border=1>
    <thead>
        <tr><td>Num</td><td>subject</td><td>content</td><td>writer</td></tr>
    </thead>
    <tbody>
        <?php
            $result = mysql_query("select * from board where {$where_clause} order by idx desc");
            while ($row = mysql_fetch_assoc($result)) {
                echo "<tr>";
                echo "<td>{$row['idx']}</td>";
                echo "<td>{$row['subject']}</td>";
                echo "<td>{$row['content']}</td>";
                echo "<td>{$row['writer']}</td>";
                echo "</tr>";
            }
        ?>
    </tbody>
    <tfoot>
        <tr><td colspan=4>
            <form method="">
                <select name="search_cols">
                    <option value="subject" selected>subject</option>
                    <option value="content">content</option>
                    <option value="content|content">subject, content</option>
                    <option value="writer">writer</option>
                </select>
                <input type="text" name="keyword" />
                <input type="radio" name="operator" value="or" checked /> or &nbsp;&nbsp;
                <input type="radio" name="operator" value="and" /> and
                <input type="submit" value="SEARCH" />
            </form>
        </td></tr>
    </tfoot>
</table>
<br />
<a href="./?view-source">view-source</a><br />
```

<br>

## Solution
---

코드 상 취약한 부분은 아래와 같다.

<br>

```php
parse_str($_SERVER['QUERY_STRING']); 

$cols = explode('|',$search_cols); 

foreach($cols as $col) { 
    $col = preg_match('/^(subject|content|writer)$/isDU',$col) ? $col : ''; 
    if($col) { 
        $query_parts = $col . " like '%" . $keyword . "%'"; 
    } 

    if($query_parts) { 
        $where_clause .= $query_parts; 
        $where_clause .= ' '; 
        $where_clause .= $operator; 
        $where_clause .= ' '; 
        $query_parts = ''; 
    } 
} 
```

<br>

우리는 쿼리로 search_cols를 보내주는데, 이 값은 explode 함수를 통해 cols에 담긴다.

cols에 값이 들어있으면 정규표현식으로 값을 검사해서 조건에 맞으면 col 변수에 값이 들어가고, 조건이 맞지 않으면 공백이 된다.

col 값이 있다면 query_parts에 값이 들어가고, 없다면 query_parts는 값이 없다.

query_parts 값이 없다면 where_clause 값에는 다른 값이 들어가게 된다.

<br>

우리는 where_clause에 공격 코드를 넣어줘야 하는데 keyword가 addslashes 되어 있어서 공략할 수 없다.

where_clause에 우리가 원하는 값을 넣어주려면 어떻게 해야 하는가 ?

<br>

col 값을 공백으로 만들어 준 뒤, query_parts를 쿼리로 값을 보내면 된다.
