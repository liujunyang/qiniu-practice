# 前端如何上传文件到七牛

标签（空格分隔）： 七牛 Node.js express Plupload qiniu-js-sdk

---

[TOC]

以下内容都是依据[官方链接](https://github.com/qiniu/js-sdk)进行实践时的经验
===

框架简介
---
Node.js使用express框架提供http支持。
使用express+ejs提供渲染带数据页面的服务来响应浏览器的页面请求
使用express的static中间件提供静态css js 图片资源的服务
使用express的get post响应页面中的请求并进行与七牛相关的处理
在浏览器打开的进行上传操作的交互页面中使用Plupload前端js库来进行上传，并利用Node.js作为后端，接受前端的上传，之后与七牛服务器进行交互。

具体细节
---
### 后端express服务 server.js ##
```javascript
/* 本文件为Node.js服务，后面引述时简称SERVER
 * 在根路径使用的ejs模板文件upload.html，简称 HTML
 * HTML中引用的upload.js为我们使用qiniu-js-sdk和Plupload的具体文件，简称HTMLJS
 */
var qiniu = require('qiniu');
var express = require('express');
var config =  {
    'ACCESS_KEY': 'Xxxxx-xxxxxx',
    'SECRET_KEY': '_xxxxAS--xxxxxxxxxxxxxxxx',
    'Bucket_Name': 'yxxxxxxxxd',
    'Port': 19110,
    'Uptoken_Url': 'uptoken',
    'Domain': 'http://xxxxxxxxxx.bkt.clouddn.com/'
};
var app = express();
// 使用static中间件提供静态资源服务，返回启动node服务的当前目录的静态资源
app.use(express.static(__dirname + '/'));
// ejs引擎配置
app.set('views', __dirname + '/');
app.engine('html', require('ejs').renderFile);

/* 前端qiniu js-sdk初始化时请求的url，这个路径是自定义的，可以定义为任意值。目前流程是：
 * 在HTML中使用的HTMLJS中使用了qiniu-js-sdk，引入了qiniu-js-sdk之后执行 Qiniu.uploader({...})时立
 * 即执行初始化，初始化时会使用uptoken_url的方式向SERVER请求token，而这个uptoken_url的地址是HTML中
 * 的$('#uptoken_url').val()，而'#uptoken_url'的值是ejs套HTML模板的时候用config.Uptoken_Url填
 * 充的（见下下方app.get('/', function(req, res) {...}部分），所以在上面的config中给Uptoken_Url设
 * 置了什么样的字符串值，下面处理token的接口就使用什么字符串。
 */
app.get('/uptoken', function(req, res, next) {
    // 使用七牛Node.js的sdk从七牛获得的token：new qiniu.rs.PutPolicy(config.Bucket_Name);
    var token = uptoken.token();
    res.header("Cache-Control", "max-age=0, private, must-revalidate");
    res.header("Pragma", "no-cache");
    res.header("Expires", 0);
    if (token) {
        // 返回给前端qiniu-js-sdk初始化时使用的token
        res.json({
            uptoken: token
        });
    }
});

app.get('/', function(req, res) {
    // 使用ejs模板引擎渲染HTML页面，塞domain和uptoken_url的值进去
    res.render('upload.html', {
        domain: config.Domain,
        uptoken_url: config.Uptoken_Url
    });
});

qiniu.conf.ACCESS_KEY = config.ACCESS_KEY;
qiniu.conf.SECRET_KEY = config.SECRET_KEY;

var uptoken = new qiniu.rs.PutPolicy(config.Bucket_Name);

app.listen(config.Port, function() {
    console.log('Listening on port %d', config.Port);
});

```

### HTML模板文件 upload.html ##
```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>前端上传-七牛云存储</title>
    <link rel="stylesheet" href="http://apps.bdimg.com/libs/todc-bootstrap/3.1.1-3.2.1/todc-bootstrap.min.css">
</head>
<body>
<div class="container">
    <div class="text-left col-md-12 wrapper">
        <h1 class="text-left col-md-12 ">
            资源上传到七牛
        </h1>
        <!-- qiniu-js-sdk初始化时请求token的url和上传地址都在这里填充模板时填充了 -->
        <input type="hidden" id="domain" value="<%= domain %>">
        <input type="hidden" id="uptoken_url" value="<%= uptoken_url %>">
        <ul class="tip col-md-12 text-mute">
            <li>
                <small>
                    通过 Html5 模式上传文件至七牛云存储。
                </small>
            </li>
            <li>
                <small>Html5模式大于4M文件采用分块上传。</small>
            </li>
            <li>
                <small>上传图片可查看处理效果。</small>
            </li>
            <li>
                <small>本示例限制最大上传文件100M。</small>
            </li>
        </ul>
    </div>
    <div class="body">
        <div class="col-md-12">
            <div id="container">
                <a class="btn btn-default btn-lg " id="pickfiles" href="#" >
                    <i class="glyphicon glyphicon-plus"></i>
                    <span>选择文件</span>
                </a>
            </div>
        </div>

        <div style="display:none" id="success" class="col-md-12">
            <div class="alert-success">
                队列全部文件处理完毕
            </div>
        </div>
        <div class="col-md-12 ">
            <table class="table table-striped table-hover text-left"   style="margin-top:40px;display:none">
                <thead>
                  <tr>
                    <th class="col-md-4">Filename</th>
                    <th class="col-md-2">Size</th>
                    <th class="col-md-6">Detail</th>
                  </tr>
                </thead>
                <tbody id="fsUploadProgress">
                </tbody>
            </table>
        </div>
    </div>
   
</div>

<script type="text/javascript" src="http://apps.bdimg.com/libs/jquery/2.1.4/jquery.min.js"></script>
<script type="text/javascript" src="http://apps.bdimg.com/libs/bootstrap/3.3.4/js/bootstrap.min.js"></script>
<script type="text/javascript" src="https://cdn.staticfile.org/plupload/2.1.2/plupload.full.min.js"></script>
<script type="text/javascript" src="https://cdn.staticfile.org/plupload/2.1.2/moxie.js"></script>
<script type="text/javascript" src="https://cdn.staticfile.org/plupload/2.1.2/i18n/zh_CN.js"></script>
<script type="text/javascript" src="module/ui.js"></script>
<!-- qiniu.js依赖于Plupload，所以Plupload要在qiniu.js之前引用 -->
<script type="text/javascript" src="https://cdn.staticfile.org/qiniu-js-sdk/1.0.14-beta/qiniu.js"></script>
<script type="text/javascript" src="https://cdn.staticfile.org/highlight.js/9.8.0/highlight.min.js"></script>
<script type="text/javascript" src="upload.js"></script>
</body>
</html>

```

### HTMLJS 前端JS upload.js ##
```javascript
// 前面已经引入好了下列js
/*global Qiniu */
/*global plupload */
/*global FileProgress */
/*global hljs */

$(function() {
    var uploader = Qiniu.uploader({
        runtimes: 'html5,flash,html4',
        browse_button: 'pickfiles',
        // HTML中重要的div的ID
        container: 'container',
        drop_element: 'container',
        max_file_size: '1000mb',
        dragdrop: true,
        chunk_size: '4mb',
        // qiniu-js-sdk初始化时请求token的url的值
        uptoken_url: $('#uptoken_url').val(),
        // 上传的域名地址
        domain: $('#domain').val(),
        get_new_uptoken: false,
        auto_start: true,
        log_level: 5,
        init: {
            'FilesAdded': function(up, files) {
                $('table').show();
                $('#success').hide();
                plupload.each(files, function(file) {
                    var progress = new FileProgress(file, 'fsUploadProgress');
                    progress.setStatus("等待...");
                    progress.bindUploadCancel(up);
                });
            },
            'BeforeUpload': function(up, file) {
                var progress = new FileProgress(file, 'fsUploadProgress');
                var chunk_size = plupload.parseSize(this.getOption('chunk_size'));
                if (up.runtime === 'html5' && chunk_size) {
                    progress.setChunkProgess(chunk_size);
                }
            },
            'UploadProgress': function(up, file) {
                var progress = new FileProgress(file, 'fsUploadProgress');
                var chunk_size = plupload.parseSize(this.getOption('chunk_size'));
                progress.setProgress(file.percent + "%", file.speed, chunk_size);
            },
            'UploadComplete': function() {
                $('#success').show();
            },
            'FileUploaded': function(up, file, info) {
                var progress = new FileProgress(file, 'fsUploadProgress');
                progress.setComplete(up, info);
            },
            'Error': function(up, err, errTip) {
                $('table').show();
                var progress = new FileProgress(err.file, 'fsUploadProgress');
                progress.setError();
                progress.setStatus(errTip);
            }
        }
    });

    uploader.bind('FileUploaded', function() {
        console.log('hello man,a file is uploaded');
    });
    
    // 拖拽时美化样式
    $('#container').on(
        'dragenter',
        function(e) {
            e.preventDefault();
            $('#container').addClass('draging');
            e.stopPropagation();
        }
    ).on('drop', function(e) {
        e.preventDefault();
        $('#container').removeClass('draging');
        e.stopPropagation();
    }).on('dragleave', function(e) {
        e.preventDefault();
        $('#container').removeClass('draging');
        e.stopPropagation();
    }).on('dragover', function(e) {
        e.preventDefault();
        $('#container').addClass('draging');
        e.stopPropagation();
    });
});

```
