+++
title = "记录下写cloud时前后端遇到的一些问题"
date = "2021-03-31"
+++

之前没怎么接触过前端，遇到了很多问题，记录一下。[cloud](https://www.wuyiting.cn/cloud)

### **Vue 及 element-ui 在cdn中可直接引入使用**

只不过使用上有些区别，遇到问题在搜索时需要注意。之后前端还是尝试使用工程搭建重构一版，现在仍然是不分离的状态。

### **axios的使用**

```jsx
axios.post('/cloud/api/delete', {'name': row.name})
   .then(function (res) {
      if (res.data.statusCode === 0) {
            ELEMENT.Message("delete success");
            ft.tableData.splice(index, 1);
      } else {
            ELEMENT.Message.warning("delete failed, " + res.data.msg);
      }
   })
```

### **前端下载文件**

与请求接口不同，页面上暂时用的js原生 - js原生

```jsx
window.open('/cloud/download/' + row.name);
```

- axios

### **局部ajax请求与后端拦截跳转**

之前的验证登陆并跳转至login页面是后端实现的：

```java
response.sendRedirect("/login");
```

这种方法在ajax请求的时候不会自动跳转至登陆页面，所以要在前端执行跳转。

```jsx
window.location.href = "/login";
```

### **后端上传与下载**

这里就直接贴下源码

```java
@Allow(role = "admin", redirect = false)
@PostMapping("/upload")
public Res upload(@RequestParam("file") MultipartFile file) {
   try {
      CloudFile cloudFile = new CloudFile();
      cloudFile.setName(file.getOriginalFilename());
      cloudFile.setSize(file.getSize());
      cloudFile.setIn(file.getInputStream());
      cloudService.upload(cloudFile);
      return Res.success();
   } catch (Throwable t) {
      return Res.unknown("/cloud", t.getMessage());
   }
}

@GetMapping("/download/{filename}")
public void download(@PathVariable String filename, HttpServletResponse response) {
   File file = new File(path + File.separator + filename);
   if (file.exists()) {
      response.setContentType("application/octet-stream");
      response.setHeader("Content-Disposition",
               "attachment; filename=" + URLEncoder.encode(filename, StandardCharsets.UTF_8));
      try (BufferedOutputStream out = new BufferedOutputStream(response.getOutputStream());
            BufferedInputStream in = new BufferedInputStream(new FileInputStream(file))) {

            byte[] b = new byte[1024];
            int len;
            while ((len = in.read(b)) != -1) {
               out.write(b, 0, len);
            }
      } catch (IOException e) {
            log.warn("download error!");
      }
   }
}
```

> 下载文件还有其他更方便的实现，比如ResponseEntity等