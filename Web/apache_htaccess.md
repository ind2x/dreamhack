## Apache htaccess
<hr style="border-top: 1px solid;"><br>

```
<VirtualHost *:80>
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

   <Directory /var/www/html/>
     AllowOverride All
     Require all granted
   </Directory>
</VirtualHost>
```

<br>

```html
<html>             <!-- index.php -->
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
        <div class="card-content">
        <h1 class="title">Online File Box</h1>
        <form action="upload.php" method="post" enctype="multipart/form-data">
            <div class="field">
                <div id="file-js" class="file has-name">
                    <label class="file-label">
                        <input class="file-input" type="file" name="file">
                        <span class="file-cta">
                            <span class="file-label">Choose a file...</span>
                        </span>
                        <span class="file-name">No file uploaded</span>
                    </label>
                </div>
            </div>
            <div class="control">
                <input class="button is-success" type="submit" value="submit">
            </div>
        </form>
        </div>
        </div>
        <script>
            const fileInput = document.querySelector('#file-js input[type=file]');
            fileInput.onchange = () => {
                if (fileInput.files.length > 0) {
                const fileName = document.querySelector('#file-js .file-name');
                fileName.textContent = fileInput.files[0].name;
                }
            }
        </script>
    </body>
</html>
```

<br>

```php
<?php /* upload.php */

$deniedExts = array("php", "php3", "php4", "php5", "pht", "phtml");

if (isset($_FILES)) {
    $file = $_FILES["file"];
    $error = $file["error"];
    $name = $file["name"];
    $tmp_name = $file["tmp_name"];
   
    if ( $error > 0 ) {
        echo "Error: " . $error . "<br>";
    }else {
        $temp = explode(".", $name);
        $extension = end($temp);
       
        if(in_array($extension, $deniedExts)){
            die($extension . " extension file is not allowed to upload ! ");
        }else{
            move_uploaded_file($tmp_name, "upload/" . $name);
            echo "Stored in: <a href='/upload/{$name}'>/upload/{$name}</a>";
        }
    }
}else {
    echo "File is not selected";
}
?>
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

$_FILES
: <a href="https://88240.tistory.com/12" target="_blank">88240.tistory.com/12</a>
: <a href="https://m.blog.naver.com/jeongju02/221471839853" target="_blank">m.blog.naver.com/jeongju02/221471839853</a>

<br>

?????? ??????????????? ??? ??? ????????? ```.htaccess``` ????????? ??????????????? ?????? ????????????.

????????? ?????? php ???????????? ????????? ?????? ???????????? ?????? ?????????????????? ??????.

```.htaccess``` ????????? ```AddType application/x-httpd-php .txt```??? ????????? ??? ????????? ?????? ??????.

?????? ```.txt``` ???????????? ```.php``` ???????????? ???????????? ????????? ????????? ????????? ????????? ???????????? ??? ???????????? ???????????? ??????.

<br>

+ Flag : ```DH{9aeba1a6feed3769ae0915b62db2b4872bec98c2}```

<br><br>
<hr style="border: 2px solid;">
<br><br>
