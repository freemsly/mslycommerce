all commerce project base on nopcommerce

1.Portal.MVC —— nopcommerce的简化版
Portal.MVC 简介
项目是基于MVC4+EF,带有角色，权限，用户中心及账户相关(登录,注册,修改密码，找回密码等)等基本功能。参考的开源项目 nopcommerce，这是一个电商架构的MVC项目，我对其进行了简化，之前主要是方便我自己搭建一些小的网站。包含前台和后台。
 1：github地址：https://github.com/stoneniqiu/Portal.MVC 

 2：百度云：链接：http://pan.baidu.com/s/1o65vgyY 密码：f4wo
 
 3.http://www.cnblogs.com/stoneniqiu/p/5010769.html
 2015-12-04 08:21 by stoneniqiu, 3507 阅读, 57 评论, 收藏, 编辑
Portal.MVC 简介
项目是基于MVC4+EF,带有角色，权限，用户中心及账户相关(登录,注册,修改密码，找回密码等)等基本功能。参考的开源项目 nopcommerce，这是一个电商架构的MVC项目，我对其进行了简化，之前主要是方便我自己搭建一些小的网站。包含前台和后台。
界面浏览
1.首页。这是前天晚上临时做出来的Logo和页面。不是真实案例，大家可以结合实际项目再修改。

2.登录框

2.注册框

3.邮箱密码找回，这个地方要用还需要配置邮箱服务器。

4.用户中心

 4.后台管理,使用的Matrix Admin 模板。这是一个很好用的模板，是衍生于Bootstrap。
 
 用户管理，这里excel导出用的是NPOI。
 
 不知道有没有提起园友的兴趣，下面我讲一下代码部分。
功能介绍
我没有做分太多的层级，这样比较直观，核心的部分在Niqiu.Core类库里面.分四个部分
 
Domain:包含主要的领域模型,比如User,UserRole,PermissionRecord等
Helpers:包含一些帮助类，比如xml，email
Mapping:数据映射
Services:服务部分的接口和实现
而网站部分重要的有一些可以复用的Attributes,AccountController等，比如UserLastActivityIpAttribute，会记录用户的Ip并更新最后访问时间。

下面介绍下2个我认为重要点的部分

Ninject依赖注入:

Nop中使用的是Autofac，并构建了一个强大的EnginContext管理所有的依赖注入项，在这个项目中我拿掉了这一部分，换成Ninject来完成IOC的工作。并不涉及这两个组件的优劣问题，而且我认为前者的功能更要强悍一些。Ninject是在NinjectWebCommon类中注册的.它在App_Start文件夹下。 如下：

kernel.Bind<IPermissionservice>().To<Permissionservice>();
Ninject更详细的介绍请看我的博客：Ninject在MVC5中的使用。在不能用构造函数的地方，可以用属性注入。

  [Inject]
   public IWorkContext WorkContext { get; set; }

  [Inject]
  public IUserService UserService { get; set; }
而有的Service需要用到HttpContext，这对象没有接口，是无法注册的，但可以通过HttpContextWrapper获得。

 public HttpContextBase HttpContext
        {
            get { return new HttpContextWrapper(System.Web.HttpContext.Current); }
        }
HttpContextBase 是一个抽象类，HttpContextWrapper是其派生成员。这两者都是.Net3.5之后才出现。
更多的解释大家可以看老赵的博客：为什么是HttpContextBase而不是IHttpContext

领域设计

 这个部分网上讨论的比较多，Nop是采用的单个仓库接口IRepository<T>，然后实现不同的服务。定义IDbContext，注入数据库对象。

 IRepository<T>:

 View Code
用EfRepository<T>实现这个接口

 View Code
而其中的IDbContext是自定义的数据接口

 View Code
然后注入：

  kernel.Bind<IDbContext>().To<PortalDb>().InSingletonScope();
对于和模型相关的Service内部都是注入的IRepository<T>，比如UserService。

复制代码
      private readonly IRepository<User> _useRepository;
       private readonly IRepository<UserRole> _userRoleRepository;
       private readonly  ICacheManager _cacheManager ;

       public UserService(IRepository<User> useRepository,IRepository<UserRole> userRoleRepository,ICacheManager cacheManager)
       {
           _useRepository = useRepository;
           _userRoleRepository = userRoleRepository;
           _cacheManager = cacheManager;
       }
复制代码
这样相互之间就比较干净。只依赖接口。

而数据模型都会继承BaseEntity这个对象。

 public class User : BaseEntity
    {
  //...
   }
数据映射相关的部分在Mapping中，比如UserMap

复制代码
 public class UserMap : PortalEntityTypeConfiguration<Domain.User.User>
    {
        public UserMap()
        {
            ToTable("Users");
            HasKey(n => n.Id);
            Property(n => n.Username).HasMaxLength(100);
            Property(n => n.Email).HasMaxLength(500);
            Ignore(n => n.PasswordFormat);
            HasMany(c => c.UserRoles).WithMany().Map(m => m.ToTable("User_UserRole_Mapping"));
        }
    }
复制代码
在PortalDb的OnModelCreating方法中加入。这样就会影响到数据库的设计。

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
        {
     
            modelBuilder.Configurations.Add(new UserMap());
        }
Nop的做法高级一些。反射找出所有的PortalEntityTypeConfiguration。一次性加入。这里大家可以自己去试

复制代码
  var typesToRegister = Assembly.GetExecutingAssembly().GetTypes()
            .Where(type => !String.IsNullOrEmpty(type.Namespace))
            .Where(type => type.BaseType != null && type.BaseType.IsGenericType &&
                type.BaseType.GetGenericTypeDefinition() == typeof(PortalEntityTypeConfiguration<>));
            foreach (var type in typesToRegister)
            {
                dynamic configurationInstance = Activator.CreateInstance(type);
                modelBuilder.Configurations.Add(configurationInstance);
            }
复制代码
权限管理:

默认设定了三种角色，Administrators，Admins，Employeer，他们分别配置了不同的权限，权限指定是在StandardPermissionProvider这个类中完成的，表示一个角色拥有哪些权限,Administrators拥有所有权限

复制代码
new DefaultPermissionRecord
      {
        UserRoleSystemName   = SystemUserRoleNames.Admin,
        PermissionRecords = new []
         {
            AccessAdminPanel,
            SearchOrder,
            ManageUsers,
          }
       },
复制代码
而权限验证的时候，在响应的Action上面加上AdminAuthorize和权限名称即可。

 [AdminAuthorize("ManageUsers")]
   public ActionResult Index()
    {
    }
而在其内部是通过PermissionService来验证的：

 public virtual bool HasAdminAccess(AuthorizationContext filterContext)
        {
            bool result = PermissionService.Authorize(Permission);
            return result;
        }
后台只有给用户设置角色的页面，我没做新增角色并分配权限的页面。不过这个实现起来也很简单了。如果你是更换数据库，这个时候设计到权限的初始化。调用下面的方法即可：

_permissionService.InstallPermissions(new StandardPermissionProvider());
当然前提是你先注入这个_permissionService。更多的代码细节大家可以去看源码。安装完成之后会有以下默认权限



 以上是关于工程的大体说明，欢迎大家拍砖。下面下载地址。数据文件需要附加。

 1：github地址：https://github.com/stoneniqiu/Portal.MVC 

 2：百度云：链接：http://pan.baidu.com/s/1o65vgyY 密码：f4wo

 数据库是Sql2008,附加不上，也可以自动生成。记得给自己添加用户。 默认用户名：stoneniqiu 密码 admin

