# cookie
#java/javaweb

参考：[理解Cookie和Session机制(转) - oneSong - 博客园](http://www.cnblogs.com/wsnb/p/5151620.html)
### cookie产生
	* Http协议无状态：为了提高http协议的效率，http不记录通信双方的状态；
	* cookie：服务器通过浏览器存放在用户本地磁盘的文本文件，以key/value形式存储信息（**明文存储**），服务器可进行文件读写操作；
### cookie应用场景：
	* “记住我”：用户登录时，web应用可记录用户名和密码；
	* 定制个性化页面；
	* 记录用户操作；
### cookie使用
	* 浏览器向web服务器发送请求时，会读取本地存储的cookie文件内容（包括过期时间和路径），一起发送给服务器；

### 服务器设置和获取cookie
	* 文件目录：
	1. ServletSetCookie：服务器将cookie内容保存在浏览器所在的客户端；
	2. ServletGetCookie：由于保存cookie后，浏览器发送请求会在header中带上cookie信息，所以直接从request中获取cookie即可；
	3. CookieInput.html：输入姓名和密码，提交表单至服务器；
	* 具体实现：
```java
／*CookieInput.html*／
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>cookie input page</title>
</head>
<body>
  <form name="form" action="/ServletSetCookie" method="post">
    <table>
      <tr>
        <td>用户名</td>
        <td><input type="text" name="username"></td>
      </tr>
      <tr>
        <td>密码</td>
        <td><input type="password" name="password"></td>
      </tr>
      <tr>
        <td><button type="submit" name="submit">提交</button></td>
      <td><button type="submit" name="submit">取消</button></td>
    </tr>
    </table>
  </form>
</body>
</html> 
```

```java
／* ServletSetCookie*／
package cookie;

import java.io.IOException;
import java.io.PrintWriter;
import java.text.SimpleDateFormat;
import java.util.Date;
import javax.servlet.ServletException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class ServletSetCookie extends HttpServlet {

  protected void doPost(HttpServletRequest request,
      HttpServletResponse response)
      throws ServletException, IOException {
    String output = null;
    String username = request.getParameter("username");
    String password = request.getParameter("password");
    if (username != null && !username.isEmpty()) {
      Cookie cookie1 = new Cookie("username", username);
      cookie1.setMaxAge(24 * 60 * 60 *30);

      Cookie cookie2 = new Cookie("password", password);
      cookie2.setMaxAge(24 * 60 * 60 *30);

      SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
      Cookie cookie3 = new Cookie("lastTime", sdf.format(new Date()));
      cookie3.setMaxAge(24 * 60 * 60 *30);
      response.addCookie(cookie1);
      response.addCookie(cookie2);
      response.addCookie(cookie3);
      output = "成功写入cookie。<a href = \"/ServletGetCookie\">查看cookie</a>";
    } else {
      output = "用户名为空，请重新输入.<br><a href = \"/cookieInput.html\">重新登陆</a>";
    }
    response.setContentType("text/html; charset=UTF-8");
    PrintWriter out = response.getWriter();
    out.println("<html><head>set cookie</head><body>");
    out.println(output);
    out.println("</body></html>");
    out.flush();
    out.close();
  }

  protected void doGet(HttpServletRequest request,
      HttpServletResponse response)
      throws ServletException, IOException {
    doPost(request, response);

  }
}
```

```java
／* ServletGetCookie *／
package cookie;

import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.ServletException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class ServletGetCookie extends HttpServlet {

  protected void doPost(HttpServletRequest request,
      HttpServletResponse response)
      throws ServletException, IOException {
    response.setContentType("text/html;charset=UTF-8");
    PrintWriter out = response.getWriter();
    out.println("<html><body>");
    Cookie[] cookies = request.getCookies();
    out.println();
    for (Cookie cookie : cookies) {
      if (cookie != null) {
        if (cookie.getName().equals("username")) {
          out.println("用户名" + cookie.getValue());
        }
        if (cookie.getName().equals("password")) {
          out.println("密码：" + cookie.getValue());
        }
        if (cookie.getName().equals("lastTime")) {
          out.println("上次登陆时间" + cookie.getValue());
        }
      }
    }
    out.print("</body></html>");
    out.flush();
    out.close();
  }

  protected void doGet(HttpServletRequest request,
      HttpServletResponse response)
      throws ServletException, IOException {
    doPost(request, response);
  }
}
```
