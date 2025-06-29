# Blog
使用golang的gin和gorm作为基本技术栈，搭建一个属于自己的个人博客系统

这篇笔记主要想记录一下我搭建个人博客系统的一个流程，并且巩固我学到的gorm+gin两个web框架
首先我们的个人博客需要做到前后端分离，所以我打算先从后端开始写起！
1.项目初始化💎
在我们喜欢的地方创建一个属于我们的个人博客项目文件夹，在这里我使用的是vscode进行开发
如果大家没有使用过vscode，可以在b站上搜索相关的教程配置好go的相关环境再进行哦。
创建好了之后切换到我们的项目文件夹下👻
cd <your_project_name>
切换好了之后进行项目的初始化,在命令行中输入以下代码
go mod init <模块名>   #例如 github.com/yourname/project

关于go mod这个项目依赖管理工具的介绍，可以看我关于它写的相关有道云笔记哦
📦 Go Modules (go mod) 简介
初始化完成之后进行必要库的安装，包括Gin框架，GORM以及数据库驱动（如MYSQL或者SQLite）
我在这里使用的是MYSQL，启动的工具使用的是phpstudy，个人觉得是一款很好用的软件，推荐大家下载
🔗:Windows版phpstudy下载 - 小皮面板(phpstudy)


下载好之后运行phpstudy就可以看到他已经自动给我们安装好了一个MySQL5.7.26版本的数据库了，点击启动就可以直接启动数据库了，在软件管理处也可以下载我们需要的其他软件。


数据库栏可以进行数据库的创建，已经修改root密码
开启数据库之后开始安装我们需要用到的库
go get -u github.com/gin-gonic/gin
go get -u gorm.io/gorm
go get gorm.io/driver/mysql #需要用哪个选哪个，我这里选MySQL
go get gorm.io/driver/postgres #需要用哪个选哪个
go get gorm.io/driver/sqlite #需要用哪个选哪个
go get gorm.io/driver/sqlserver #需要用哪个选哪个
如果嫌弃下载速度慢可以先配置以下GOPROXY进行加速
go env -w GOPROXY=https://goproxy.cn,direct
连接数据库我用的时vscode的MYSQL管理插件，大家根据自己的需求自行选择即可
当相关库都下载完毕，并且数据库正常时就可以进行下一步了

2.📝数据库设计与模型定义
🗂️ 数据库表结构设计
1. Users 表（用户信息）
字段名类型说明约束iduint主键，自增PRIMARY KEYusernamevarchar(50)用户名，唯一UNIQUE, NOT NULLpasswordvarchar(100)加密后的密码NOT NULLemailvarchar(100)邮箱，唯一UNIQUE, NOT NULLcreated_attimestamp创建时间DEFAULT CURRENT_TIMESTAMPupdated_attimestamp更新时间DEFAULT CURRENT_TIMESTAMP ON UPDATE
2. Posts 表（博客文章）
字段名类型说明约束iduint主键，自增PRIMARY KEYtitlevarchar(100)文章标题NOT NULLcontenttext文章内容NOT NULLuser_iduint关联 users 表的作者FOREIGN KEY (user_id) REFERENCES users(id)created_attimestamp创建时间DEFAULT CURRENT_TIMESTAMPupdated_attimestamp更新时间DEFAULT CURRENT_TIMESTAMP ON UPDATE
3. Comments 表（文章评论）
字段名类型说明约束iduint主键，自增PRIMARY KEYcontenttext评论内容NOT NULLuser_iduint关联 users 表的评论者FOREIGN KEY (user_id) REFERENCES users(id)post_iduint关联 posts 表的文章FOREIGN KEY (post_id) REFERENCES posts(id)created_attimestamp创建时间DEFAULT CURRENT_TIMESTAMP
🛠️ GORM 模型定义（Go 结构体）
package models

//包含了之后需要用到的库
import (
    "log"
    "net/http"
    "time"
    "github.com/dgrijalva/jwt-go"
    "github.com/gin-gonic/gin"
    "golang.org/x/crypto/bcrypt"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

// User 用户模型
type User struct {
    gorm.Model           // 内嵌 gorm.Model，包含 ID、CreatedAt、UpdatedAt、DeletedAt
    Username string `gorm:"type:varchar(50);unique;not null"`
    Password string `gorm:"type:varchar(100);not null"`
    Email    string `gorm:"type:varchar(100);unique;not null"`
    Posts    []Post // 一对多关系：用户拥有多篇文章
    Comments []Comment // 一对多关系：用户有多条评论
}

// Post 文章模型
type Post struct {
    gorm.Model
    Title    string `gorm:"type:varchar(100);not null"`
    Content  string `gorm:"type:text;not null"`
    UserID   uint   // 外键关联 User 模型
    User     User   `gorm:"foreignKey:UserID"` // 属于关系：文章属于用户
    Comments []Comment // 一对多关系：文章有多条评论
}

// Comment 评论模型
type Comment struct {
    gorm.Model
    Content string `gorm:"type:text;not null"`
    UserID  uint   // 外键关联 User 模型
    PostID  uint   // 外键关联 Post 模型
    User    User   `gorm:"foreignKey:UserID"` // 属于关系：评论属于用户
    Post    Post   `gorm:"foreignKey:PostID"` // 属于关系：评论属于文章
}
🔗 关联关系说明
- 一对多：
- 一个 User 拥有多篇 Post（User.Posts 字段）。
- 一个 Post 拥有多条 Comment（Post.Comments 字段）。
- 属于（Belongs To）：
- Post 属于 User（通过 Post.UserID 外键）。
- Comment 属于 User 和 Post（通过 UserID 和 PostID）。


--------------------------------------------------------------------------------
3.🌠后端代码编写
用户
我们先来实现这几个功能
- 为了方便日后的维护和管理，咱们使用日志库记录系统的运行信息和错误信息，对可能出现的错误进行统一处理，如数据库连接错误、用户认证失败、文章或评论不存在等，返回合适的 HTTP 状态码和错误信息。
- 实现用户注册和登录功能，用户注册时需要对密码进行加密存储，登录时验证用户输入的用户名和密码。,
- 使用 JWT（JSON Web Token）实现用户认证和授权，用户登录成功后返回一个 JWT，后续的需要认证的接口需要验证该 JWT 的有效性。
统一错误响应
// 统一错误响应
func respondWithError(c *gin.Context, code int, message string, err error) {
    if err != nil {
        log.Printf("Error: %s | Detail: %v", message, err)
    } else {
        log.Printf("Error: %s", message)
    }
    c.JSON(code, gin.H{"error": message})
}
用户注册
//这里是我们的注册方法
func register(c *gin.context){
    //首先定义一个user结构体去接收前端传来的信息
    var user User
    //这里我们使用gin自带的ShouldBindJSON函数去将前端传来的JSON格式的用户信息自动映射到结构体里面去
    //并且咱们这里使用if+err的形式，如果有任何错误那么直接结束函数操作，并且使用我们错误处理函数去记录
    if err := c.ShouldBindJSON(&user); err != nil{
            rerespondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
            return
    }
    
    //如果上一步没有出现问题，那么就进入下一步,将用户的密码进行hash加密
    //这里我们使用bcrypt库里面的GenerateFromPassword方法，需要传两个参数
    //第一个是我们的密码,类型是[]byte,第二个是我们的加密因子，类型为int，通常输入为bcrypt.DefaultCost
    //返回值也有两个，一个是加密后的密码，类型是[]byte，另一个是err
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
    if err != nil {
        respondWithError(c, http.StatusInternalServerError, "密码加密失败", err)
        return
    }
    //因为返回值的类型是[]byte,所以我们这里将他的类型转换为string，方便我们存入数据库
    user.Password = string(hashedPassword)
    //接下来就是存入数据库的操作了，还是用if+err的形式
    if err := db.Create(&uer).Error; err != nil{
        respondWithError(c, http.StatusInternalServerError, "用户创建失败", err)
        return
    }
    //如果都正常的话证明创建成功，咱们通过c.JSON向前端传递正确的http状态码和信息
    c.JSON(http.StatusCreated, gin.H{"message": "User registered successfully"})
    }
用户登录
// 用户登录方法
func Login(c *gin.Context) {
    //首先定义一个user结构体用来存前端传来的JSON数据
    var user User
    //这里ShouldBindJSON函数上面有讲，这里不展开说明
    if err := c.ShouldBindJSON(&user); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }
    //这里再定义一个user结构体用来存数据库中的user信息
    var storedUser User
    //这里使用条件判断找前端传过来的用户名是否在数据库中存在，如果不存在则返回不存在的报错信息
    if err := db.Where("username = ?", user.Username).First(&storedUser).Error; err != nil {
        respondWithError(c, http.StatusUnauthorized, "用户名不存在", err)
        return
    }
    //如果用户名存在那么我们就使用bcrypt库中的CompareHashAndPassword函数进行密码的对比
    //这个函数有两个参数，类型都是[]byte，第一个参数是数据库中hash过来存储的密码，第二个是前端传来的密码
    //如果两个密码一样则返回nil，如果不相等则报错直接退出
    if err := bcrypt.CompareHashAndPassword([]byte(storedUser.Password), []byte(user.Password)); err != nil {
        respondWithError(c, http.StatusUnauthorized, "用户名或密码错误", err)
        return
    }
//这里使用的是jwt的一个方法
//作用：创建一个新的 JWT 对象，并指定签名算法和载荷（Claims）
//你可以理解为“生成一个待签名的 Token 对象”，但此时还没有生成最终的字符串。
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "id":       storedUser.ID,
        "username": storedUser.Username,
        "exp":      time.Now().Add(time.Hour * 24).Unix(),
        //这一行是JWT有效期，从创建的时候开始，有效时间就是time.Hour * 24，也就是一天
    })
//这里是jwt的一个方法，用来生成Token的。
//作用：用指定的密钥对上面生成的 Token 对象进行签名，得到最终可以传递和验证的 JWT 字符串。
//你可以理解为“把 Token 对象变成字符串，并加密签名”。
//其中的参数我写的是"your_secret_key"，就是你的JWT签名密钥，你可以自己定义，只要保证前后端验证时用的是同一个密钥即可。
    tokenString, err := token.SignedString([]byte("your_secret_key"))
    if err != nil {
        respondWithError(c, http.StatusInternalServerError, "Token生成失败", err)
        return
    }
//如果都没有问题那么就向前端传递正确的http状态码和信息
    c.JSON(http.StatusOK, gin.H{
        "message": "User Login successfully",
        "token":   tokenString,
    })
}
用户认证（中间件）
// 定义一个JWT校验的中间件来判断token是否有效
//中间件的固定写法就是这样，大家可以记一下，这里的方法我就不细讲了，大家不需要太记住只有一个需要注意的点
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenString := c.GetHeader("Authorization")
        if tokenString == "" {
            respondWithError(c, http.StatusUnauthorized, "缺少token", nil)
            c.Abort()
            return
        }
        if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
            tokenString = tokenString[7:]
        }
        token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
            return []byte("your_secret_key"), nil
        })//这里的JWT签名密钥要跟之前你自己的一样我这里是"your_secret_key"
        if err != nil || !token.Valid {
            respondWithError(c, http.StatusUnauthorized, "无效的token", err)
            c.Abort()
            return
        }
        c.Next()
    }
}
--------------------------------------------------------------------------------

文章
- 实现文章的创建功能，只有已认证的用户才能创建文章，创建文章时需要提供文章的标题和内容
- 实现文章的读取功能，支持获取所有文章列表和单个文章的详细信息。
- 实现文章的更新功能，只有文章的作者才能更新自己的文章。
- 实现文章的删除功能，只有文章的作者才能删除自己的文章
创建文章
// 实现文章的创建功能，只有已认证的用户才能创建文章，创建文章时需要提供文章的标题和内容。（配合中间件使用）
func CreatePost(c *gin.Context) {
    var p Post
    if err := c.ShouldBindJSON(&p); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }
    if p.Title == "" || p.Content == "" {
        respondWithError(c, http.StatusBadRequest, "文章内容或者标题为空，请添加文章内容或者标题", nil)
        return
    }
    //这里先获取一下前端传来的token
    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    //将token进行解密，得到最后我们需要的用户id
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid := uint(claims["id"].(float64))
        p.UserID = uid //如果最终token有效，则将post的userid设定为token中的userid
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }
//最后进行文章的创建
    if err := db.Create(&p).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "创建文章失败", err)
        return
    }
    c.JSON(http.StatusCreated, gin.H{"message": "文章创建成功", "post": p})
}
读取全部文章、读取单个文章
// 获取所有文章列表，这里很简单就不做过多的讲解
func GetPosts(c *gin.Context) {
    var posts []Post
    if err := db.Preload("User").Find(&posts).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "获取文章列表失败", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"posts": posts})
}
// 获取单个文章详情，使用preload就可以了，还可以嵌套使用
func GetPostDetail(c *gin.Context) {
    id := c.Param("id")
    var post Post
    if err := db.Preload("User").Preload("Comments.User").First(&post, id).Error; err != nil {
        respondWithError(c, http.StatusNotFound, "文章不存在", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"post": post})
}
更新文章
func UpdatePost(c *gin.Context) {
    //首先获取从前端传来的需要更改的文章id
    id := c.Param("id")
    var post Post
    //这里判断一下文章存不存在
    if err := db.First(&post, id).Error; err != nil {
        respondWithError(c, http.StatusNotFound, "文章不存在", err)
        return
    }
    //然后拿到签到传来的token，得到最终的uid，再进行对比是否是文章的创建者
    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    var uid uint
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid = uint(claims["id"].(float64))
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }
    if post.UserID != uid {
        respondWithError(c, http.StatusForbidden, "无权限修改此文章", nil)
        return
    }
    //这里先自定义一个结构体去存前端传来的更新过后title和content
    var req struct {
        Title   string `json:"title"`
        Content string `json:"content"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }
    //如果标题和内容不为空，那么将post的标题和内容进行更换
    if req.Title != "" {
        post.Title = req.Title
    }
    if req.Content != "" {
        post.Content = req.Content
    }
    //最后进行更新的保存
    if err := db.Save(&post).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "更新失败", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "更新成功", "post": post})
}
删除文章
func DeletePost(c *gin.Context) {
    id := c.Param("id")
    var post Post
    if err := db.First(&post, id).Error; err != nil {
        respondWithError(c, http.StatusNotFound, "文章不存在", err)
        return
    }
    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    var uid uint
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid = uint(claims["id"].(float64))
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }
    if post.UserID != uid {
        respondWithError(c, http.StatusForbidden, "无权限删除此文章", nil)
        return
    }
    if err := db.Delete(&post).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "删除失败", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "删除成功"})
}
以上就是我们文章的全部功能
--------------------------------------------------------------------------------

评论
- 实现评论的创建功能，已认证的用户可以对文章发表评论。,
- 实现评论的读取功能，支持获取某篇文章的所有评论列表。
创建评论
func CreateComment(c *gin.Context) {
    // 只绑定 content 和 post_id，避免 user_id 被前端伪造
    var req struct {
        Content string `json:"content"`
        PostID  uint   `json:"post_id"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }
    // 从 token 获取用户ID（已通过中间件校验）
    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    var uid uint
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid = uint(claims["id"].(float64))
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }
    comment := Comment{
        Content: req.Content,
        PostID:  req.PostID,
        UserID:  uid,
    }
    if err := db.Create(&comment).Error; err != nil {
        respondWithError(c, http.StatusBadRequest, "评论创建失败", err)
        return
    }
    c.JSON(http.StatusCreated, gin.H{"message": "评论创建成功", "comment": comment})
}
再就是评论的读取功能，这个其实直接通过获取单个文章详情功能实现，我就没有再单独写了
以上就是后端逻辑代码的所有内容啦
--------------------------------------------------------------------------------
4.👑主函数代码编写
主函数

func main() {
    //首先配置好我们的这个dsn格式是用户名，密码
    //localhost:3306是数据库运行的地址和端口，Blog是我数据库
    dsn := "Hickerzed:1304399070hxz@tcp(localhost:3306)/Blog?charset=utf8mb4&parseTime=True&loc=Local"
    //这里直接连接我们的数据库，我这里用的是mysql，如果你用的其他的可以进行更改，并且使用对应的库
    db, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("连接数据库失败: %v", err)
    }
    //这里进行我们数据库表的创建
    if err := db.AutoMigrate(&User{}, &Post{}, &Comment{}); err != nil {
        log.Fatalf("数据库迁移失败: %v", err)
    }
    //这里来初始化我们的路由，一般都是这样写
    r := gin.Default()
    //这里我分了三个路由组，分别是user，post和comment方便后续的测试
    //对于需要使用身份认证的方法，我们直接在需要用的方法中去添加我们的中间件即可
    //如果整个路由组都要用的话，我以user为例子，那么直接user.Use(AuthMiddleware())即可
    user := r.Group("/api/user")
    {
        user.POST("/register", Register)
        user.POST("/login", Login)
    }

    post := r.Group("/api/post")
    {
        post.POST("/createpost", AuthMiddleware(), CreatePost)
        post.GET("/getposts", GetPosts)
        post.GET("/:id", GetPostDetail)
        post.PUT("/:id", UpdatePost)
        post.DELETE("/:id", DeletePost)
    }

    comment := r.Group("/api/comment")
    {
        comment.POST("/createcomment", AuthMiddleware(), CreateComment)
        comment.GET("/post/:id", GetPostDetail)
    }
    //启动我们的路由，并且我们可以自定义启动的端口，默认是运行在8080上面
    err := r.Run()
    if err != nil {
        log.Fatalf("启动路由失败: %v", err)
        return
    }
}


--------------------------------------------------------------------------------
5.🗝️利用Postman进行api的测试
API测试

Register
方法是POST，url是http://localhost:8080/api/user/register，示例为
{
  "username": "Hickerzed",
  "password": "1304399070hxz",
  "email": "3109879624@qq.com"
}
点击send如果成功的话则如我截图所示



Login
方法是POST，url是http://localhost:8080/api/user/register，示例为
{
  "username": "Hickerzed",
  "password": "1304399070hxz"
}
如果成功的话会返回用户注册成功，并且返回一个生成的token，这个token咱们需要保存下来之后会用到



创建文章
方法是POST，url是http://localhost:8080/api/post/createpost，示例为
Header中key为 Authorization，value为Bearer <你生成的token>(记得不要加<>)
{
  "title": "Hickerzed的文章",
  "content": "乱七八糟"
}
成功的示例截图



获取所有文章
方法是GET，url:http://localhost:8080/api/post/getposts，成功截图如下



获取单个文章
方法是GET，URL:http://localhost:8080/api/post/:id，示例就是Params中的key为id的一行，将value改成你想要查的文章id
成功示例截图如下



更新文章
方法是PUT，URL:http://localhost:8080/api/post/:id
示例就是想更新哪一个就将Params中的key为id的一行的value改成你想要更新的文章id
Header中key为 Authorization，value为Bearer <你生成的token>(记得不要加<>)
成功示例图



删除文章
方法是DELETE，URL是http://localhost:8080/api/post/:id
示例就是想删除哪一个就将Params中的key为id的一行的value改成你想要删除的文章id
Header中key为 Authorization，value为Bearer <你生成的token>(记得不要加<>)
成功示例



创建评论
方法是POST，URL：http://localhost:8080/api/comment/createcomment，示例如下
Header中key为 Authorization，value为Bearer <你生成的token>(记得不要加<>)
{
  "content": "这是Hickerzed的评论",
  "post_id": 1
}
成功的截图如下



获取某个文章的所有评论详细
方法为GET，URL：http://localhost:8080/api/comment/post/:id，示例的话参考获取某个文章的详细信息
成功的截图如下


看到这里之后那么恭喜你，自此你已经成功完成了后端的基本逻辑以及各个端口的测试🥳
--------------------------------------------------------------------------------
接下来是整个后端的代码
--------------------------------------------------------------------------------
package main

import (
    "log"
    "net/http"
    "time"

    "github.com/dgrijalva/jwt-go"
    "github.com/gin-gonic/gin"
    "golang.org/x/crypto/bcrypt"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

// 定义两个全局变量
var db *gorm.DB
var err error

type User struct {
    gorm.Model           // 内嵌 gorm.Model，包含 ID、CreatedAt、UpdatedAt、DeletedAt
    Username   string    `gorm:"type:varchar(50);unique;not null"`
    Password   string    `gorm:"type:varchar(100);not null"`
    Email      string    `gorm:"type:varchar(100);unique;not null"`
    Posts      []Post    // 一对多关系：用户拥有多篇文章
    Comments   []Comment // 一对多关系：用户有多条评论
}

// Post 文章模型
type Post struct {
    gorm.Model
    Title    string    `gorm:"type:varchar(100);not null"`
    Content  string    `gorm:"type:text;not null"`
    UserID   uint      // 外键关联 User 模型
    User     User      `gorm:"foreignKey:UserID"` // 属于关系：文章属于用户
    Comments []Comment // 一对多关系：文章有多条评论
}

// Comment 评论模型
type Comment struct {
    gorm.Model
    Content string `gorm:"type:text;not null"`
    UserID  uint   // 外键关联 User 模型
    PostID  uint   // 外键关联 Post 模型
    User    User   `gorm:"foreignKey:UserID"` // 属于关系：评论属于用户
    Post    Post   `gorm:"foreignKey:PostID"` // 属于关系：评论属于文章
}

// 统一错误响应
func respondWithError(c *gin.Context, code int, message string, err error) {
    if err != nil {
        log.Printf("Error: %s | Detail: %v", message, err)
    } else {
        log.Printf("Error: %s", message)
    }
    c.JSON(code, gin.H{"error": message})
}

// 用户注册方法
func Register(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }

    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
    if err != nil {
        respondWithError(c, http.StatusInternalServerError, "密码加密失败", err)
        return
    }
    user.Password = string(hashedPassword)

    if err := db.Create(&user).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "用户创建失败", err)
        return
    }
    c.JSON(http.StatusCreated, gin.H{"message": "User registered successfully"})
}

// 用户登录方法
func Login(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }

    var storedUser User
    if err := db.Where("username = ?", user.Username).First(&storedUser).Error; err != nil {
        respondWithError(c, http.StatusUnauthorized, "用户名不存在", err)
        return
    }

    if err := bcrypt.CompareHashAndPassword([]byte(storedUser.Password), []byte(user.Password)); err != nil {
        respondWithError(c, http.StatusUnauthorized, "用户名或密码错误", err)
        return
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "id":       storedUser.ID,
        "username": storedUser.Username,
        "exp":      time.Now().Add(time.Hour * 24).Unix(),
    })

    tokenString, err := token.SignedString([]byte("your_secret_key"))
    if err != nil {
        respondWithError(c, http.StatusInternalServerError, "Token生成失败", err)
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "message": "User Login successfully",
        "token":   tokenString,
    })
}

// 定义一个JWT校验的中间件来判断token是否有效
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenString := c.GetHeader("Authorization")
        if tokenString == "" {
            respondWithError(c, http.StatusUnauthorized, "缺少token", nil)
            c.Abort()
            return
        }
        if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
            tokenString = tokenString[7:]
        }
        token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
            return []byte("your_secret_key"), nil
        })
        if err != nil || !token.Valid {
            respondWithError(c, http.StatusUnauthorized, "无效的token", err)
            c.Abort()
            return
        }
        c.Next()
    }
}

// 实现文章的创建功能，只有已认证的用户才能创建文章，创建文章时需要提供文章的标题和内容。（配合中间件使用）
func CreatePost(c *gin.Context) {
    var p Post
    if err := c.ShouldBindJSON(&p); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }
    if p.Title == "" || p.Content == "" {
        respondWithError(c, http.StatusBadRequest, "文章内容或者标题为空，请添加文章内容或者标题", nil)
        return
    }

    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid := uint(claims["id"].(float64))
        p.UserID = uid
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }

    if err := db.Create(&p).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "创建文章失败", err)
        return
    }
    c.JSON(http.StatusCreated, gin.H{"message": "文章创建成功", "post": p})
}

// 获取所有文章列表
func GetPosts(c *gin.Context) {
    var posts []Post
    if err := db.Preload("User").Find(&posts).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "获取文章列表失败", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"posts": posts})
}

// 获取单个文章详情
func GetPostDetail(c *gin.Context) {
    id := c.Param("id")
    var post Post
    if err := db.Preload("User").Preload("Comments.User").First(&post, id).Error; err != nil {
        respondWithError(c, http.StatusNotFound, "文章不存在", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"post": post})
}

func UpdatePost(c *gin.Context) {
    id := c.Param("id")
    var post Post
    if err := db.First(&post, id).Error; err != nil {
        respondWithError(c, http.StatusNotFound, "文章不存在", err)
        return
    }

    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    var uid uint
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid = uint(claims["id"].(float64))
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }

    if post.UserID != uid {
        respondWithError(c, http.StatusForbidden, "无权限修改此文章", nil)
        return
    }

    var req struct {
        Title   string `json:"title"`
        Content string `json:"content"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }
    if req.Title != "" {
        post.Title = req.Title
    }
    if req.Content != "" {
        post.Content = req.Content
    }
    if err := db.Save(&post).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "更新失败", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "更新成功", "post": post})
}

func DeletePost(c *gin.Context) {
    id := c.Param("id")
    var post Post
    if err := db.First(&post, id).Error; err != nil {
        respondWithError(c, http.StatusNotFound, "文章不存在", err)
        return
    }

    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    var uid uint
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid = uint(claims["id"].(float64))
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }

    if post.UserID != uid {
        respondWithError(c, http.StatusForbidden, "无权限删除此文章", nil)
        return
    }

    if err := db.Delete(&post).Error; err != nil {
        respondWithError(c, http.StatusInternalServerError, "删除失败", err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "删除成功"})
}

func CreateComment(c *gin.Context) {
    // 只绑定 content 和 post_id，避免 user_id 被前端伪造
    var req struct {
        Content string `json:"content"`
        PostID  uint   `json:"post_id"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        respondWithError(c, http.StatusBadRequest, "参数绑定失败", err)
        return
    }

    // 从 token 获取用户ID（已通过中间件校验）
    tokenString := c.GetHeader("Authorization")
    if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
        tokenString = tokenString[7:]
    }
    token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("your_secret_key"), nil
    })
    var uid uint
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        uid = uint(claims["id"].(float64))
    } else {
        respondWithError(c, http.StatusUnauthorized, "无效的token", nil)
        return
    }

    comment := Comment{
        Content: req.Content,
        PostID:  req.PostID,
        UserID:  uid,
    }

    if err := db.Create(&comment).Error; err != nil {
        respondWithError(c, http.StatusBadRequest, "评论创建失败", err)
        return
    }
    c.JSON(http.StatusCreated, gin.H{"message": "评论创建成功", "comment": comment})
}

func main() {
    dsn := "Hickerzed:1304399070hxz@tcp(localhost:3306)/Blog?charset=utf8mb4&parseTime=True&loc=Local"
    db, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("连接数据库失败: %v", err)
    }
    if err := db.AutoMigrate(&User{}, &Post{}, &Comment{}); err != nil {
        log.Fatalf("数据库迁移失败: %v", err)
    }
    r := gin.Default()

    user := r.Group("/api/user")
    {
        user.POST("/register", Register)
        user.POST("/login", Login)
    }

    post := r.Group("/api/post")
    {
        post.POST("/createpost", AuthMiddleware(), CreatePost)
        post.GET("/getposts", GetPosts)
        post.GET("/:id", GetPostDetail)
        post.PUT("/:id", UpdatePost)
        post.DELETE("/:id", DeletePost)
    }

    comment := r.Group("/api/comment")
    {
        comment.POST("/createcomment", AuthMiddleware(), CreateComment)
        comment.GET("/post/:id", GetPostDetail)
    }

    err := r.Run()
    if err != nil {
        log.Fatalf("启动路由失败: %v", err)
        return
    }
}
