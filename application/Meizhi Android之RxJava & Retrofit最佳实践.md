![](https://wallcat.imgix.net/os43HKoEfD.jpg?crop=entropy&dpr=2&h=800&ixjsv=2.1.0&q=70&w=1400)

## 零、背景
比起阅读枯燥的技术文档，独自苦苦摸索新技术的基本用法，还有一种更好更快速也更有效的提高自身技术的方法，那就是阅读学习优质的开源项目，通过仿写、练习最终达到理解，潜移默化提升自身编程技能。

《带你学开源项目》系列将带领你深入阅读及分析当前流行的一些开源项目，并针对其中采用的新技术与精妙之处进行细致的阐述，以期让你快速掌握Android开发中的多种强大技能点。

## 一、本期开源项目Meizhi Android
本次的开源项目选择了[Meizhi Android](https://github.com/drakeet/Meizhi)，本文主要介绍该项目中采用的`RxJava`、`Retrofit`两种技术，这二者在Android开发者中非常流行，不仅能够`优美地处理异步回调`，而且能`提高代码的性能和稳定性`。而Meizhi Android中较好的覆盖了二者的多种应用场景，能够给多数开发者一个全面的学习。

下面本人会`对原项目的代码进行详细的介绍`，同时为了读者看的清楚其中的逻辑关系，可能会做一定调整以帮助读者理解，比如把lambda表达式还原成普通java函数形式，以避免很多读者对lambda并不熟悉。

##二、原项目分析
###0. clone项目到本地
第一步当然是把项目clone下来，编译，运行。有兴趣的同学可以执行这一步。
###1. 添加`Stetho`抓包工具
首先，由于我们要分析retrofit，所以为了查看app的网络请求，有兴趣的同学可以手动在代码里添加[Stetho](http://facebook.github.io/stetho/)。`Stetho`是Facebook推出的一款黑科技，能够在chrome里轻松查看app所有的网络请求，比起iOS需要装个[Charles](https://www.charlesproxy.com/)查看http请求方便多咯。

![Stetho使用场景](http://upload-images.jianshu.io/upload_images/281665-50854bb575db05f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###2. Retrofit结构
从下图我们可以看到，首页里有很多card，每一个card里有两个元素：`妹纸图片`， `描述文字`，具体UI实现我们不在乎，只要明白一点，这两个元素数据是来自于两个不同的api。其中，`妹纸图片`来自于`http://gank.io/api/data/福利/10`;`描述文字`来自于`http://gank.io/api/data/休息视频/10`。

app中为了请求网络数据，采用了[Retrofit](http://square.github.io/retrofit/)。具体关于retrofit如何配置请各位参考官网，这里只讲解如何使用`Retrofit`。

该项目中主要创建了以下几个类来实现`Retrofit`结构，大家可以作为参考用于自己的项目中。

 #####i. `GankApi`：这个类用来定义相关的`http`接口，这是符合retrofit规范的定义形式，每一个api返回的为`Observable<T>`格式结果，方便`RxJava`进行进一步处理。
    @GET("/data/福利/{page}") Observable<MeizhiList> getMeizhiList(@Path("page") int page);
    @GET("/data/休息视频/{page}") Observable<GankVideoList> getGankVideoList(@Path("page") int page);
#####ii. `DrakeetRetrofit`：这个类用来对`Retrofit`进行相关配置并生成`GankApi`实例`gankApi`
    OkHttpClient client = new OkHttpClient();
    RestAdapter.Builder builder = new RestAdapter.Builder();
    builder.setClient(new OkClient(client))
          .setLogLevel(RestAdapter.LogLevel.FULL) 
          .setEndpoint("http://gank.io/api")
          .setConverter(new GsonConverter(gson));
    RestAdapter gankRestAdapter = builder.build();
    GankApi gankApi = gankRestAdapter.create(GankApi.class);

    public GankApi getGankApi() {    
        return gankApi;
    }
#####iii. `DrakeetFactory`： 这个类用来对外生成单例`GankApi`实例，为确保`GankApi`实例只生成一次。
    public static GankApi getGankApi() {    
        synchronized (monitor) {        
           if (sGankApi == null) {            
              sGankApi = new DrakeetRetrofit().getGankApi();        
           }       
           return sGankApi;    
        }
    }

所以，在实际应用场景中，比如我们想要发起一个http请求来获取`福利`数据，那么我们可以采用以下方式：

    GankApi gankApi = DrakeetFactory.getGankApi();
    Observable<MeizhiList> meizhiList = gankApi. getMeizhiList(10);


![首页.png](http://upload-images.jianshu.io/upload_images/281665-2129d871aff9a884.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###3. 首页的RxJava的实现
既然我们已经把网络框架搭建好了，那么可以开始从服务器获取数据并显示了。我们首先看首页的数据。下面，我来对首页数据进行分析，一步步推出所需要的RxJava表达式。

上面已经介绍过，每一个card里有两部分数据：`妹纸图片`(红色方框)和`描述文本`(绿色方框)。

- `妹纸图片`数据来自于`"/data/福利/{page}"`这个api，该api会返回妹纸图片的url；
- `描述文本`来自于`"/data/休息视频/{page}"`这个api，该api会返回休息视频及相关描述信息，card里会把描述信息显示出来；
- 两个api均可以携带`page`字段，即一次请求可以获得多个数据。如我们在`"/data/福利/{page}"`里设置`page=10`，那么我们一次请求可以得到10条`福利`数据，即`10张妹纸图片url`；
- 由于我们一次可以获得多张妹纸图片url和多个视频信息，那我们就需要把`二者进行合并`，即`单拎出来一张妹纸图片和一个视频信息组装成一个card`。然后按这种方式生成其他的card。

小结一下，根据以上描述，假如我们把两个api的page都设置为`10`，那么两个请求同时发出去后，我们能得到`10张妹纸图片url`（如`http://img.com/1.png`, `http://img.com/2.png`, ...）和`10个视频信息`(如`舌尖上的中国`, `星际穿越`， ...)，然后我们将二者组装成`10个card所需要的数据`，放入每个card里显示即可。

好，终于可以开始动手写代码了。上面的分析看似复杂，然后只要你学会了如何分析，很快就能写出对应的RxJava代码。下面我结合RxJava的`数据流思想`和`具体操作符`来介绍实现代码。

#####i. 在网络请求数据之前，我们要创建几个数据entry对象来将获取回来的json字符串转化为object
    public class Meizhi {
        public String url;
        public Date publishDate;
    } //这是一个Meizhi对象，存储妹纸图片的url，图片描述信息和创建日期

    public class Video {
      public String desc;
      public Date publishDate;
    } //这是一个视频对象，存储视频描述信息和创建日期

    public class MeizhiList {
      public List<Meizhi> meizhiList;
    } //由于我们一次请求能获取到10个(根据`page`设置)，所以我们用MeizhiList来存储结果

    public class VideoList {
      public List<Video> videoList;
    } //原理同上，存储多个video对象

    public class MeizhiWithVideo {
      public String url;
      public String desc;
      public Date publishDate;
    }//将video信息合并入meizhi对象中

    public class MeizhiWithVideoList {
      public List<MeizhiWithVideoList> data;
    }
  
#####ii. zip: 将两个retrofit接口请求后得到的两个数据源Observable<MeizhiList>  Observable<VideoList>进行合并
我们需要把这两个数据源的数据拼接起来，所以我们可以考虑使用[zip操作符](http://reactivex.io/documentation/operators/zip.html)，该操作符可以将两个数据源发射出来的数据依次组装在一起。

比如一个`Observable数据源`依次发射出`1, 3, 5, 7`, 另一个`Observable数据源`依次发射出`a, b, c, d`，那么`zip操作符`组装后会对外发射出`1a, 3b, 5c, 7d`这样的数据。

而我们需要的正是这样。

`Observable<MeizhiList>`一次对外发射一个`MeizhiList`对象，`Observable<VideoList>`一次对外发射一个`VideoList`对象，我们将二者合并成一个`MeizhiWithVideoList`对象。然后把`MeizhiWithVideoList`对象拿给UI去进行显示即可。

所以，我们可以得到：

    Observable<MeizhiList> meizhiListObservable = gankApi.getMeizhiList(10);
    Observable<VideoList> videoListObservable = gankApi.getVideoList(10);
    Observable<MeizhiWithVideoList> meizhiWithVideoListObservable = 
    Observable.zip(meizhiListObservable, videoListObservable, this::mergeVideoWithMeizhi)

其中`mergeVideoWithMeizhi`是一个合并函数，把`video`信息与`meizhi`信息合并成新的`MeizhiWithVideo对象`。

    public MeizhiWithVideoList
    mergeVideoWithMeizhi(MeizhiList meizhiList, VideoList videoList) {//省略...}

![RxJava - zip](http://upload-images.jianshu.io/upload_images/281665-91320f2cce108a8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#####iii. 对MeizhiWithVideo对象进行排序。
在上面，我们通过合并，得到了  `Observable<MeizhiWithVideoList>`数据源，这个数据源对外发射出一个`MeizhiWithVideoList`对象，这个对象里有10个`MeizhiWithVideo`数据，我们可以对这10个数据利用它们的发布日期进行排序。

所以我们要实现以下几步：
- 先把`Observable<MeizhiWithVideoList>`数据源转化为`Observable<List<MeizhiWithVideo>>`，从对外发一个`MeizhiWithVideoList`对象变成对外发射一个`List<MeizhiWithVideo>`对象；

- 再把`Observale<List<MeizhiWithVideo>>`转化为`Observable<MeizhiWithVideo>`数据源，变成了对外发射出10个`MeizhiWithVideo`对象;

- 对这10个`MeizhiWithVideo`对象基于`publishDate`进行排序；

- 其中比较操作很耗cpu，所以我们放在`Schedulers.computation()`线程中做

代码实现：
    
    meizhiWithVideoListObservable.map(new Func1<MeizhiWithVideoList, List<MeizhiWithVideo>>() {    
          @Override    
          public List<Meizhi> call(MeizhiList meizhiList) {                
                return MeizhiWithVideoList.data;    
          }
    })
    .flatMap(new Func1<List<MeizhiWithVideo>, Observable<MeizhiWithVideo>>() {    
          @Override    
          public Observable<MeizhiWithVideo> call(List<MeizhiWithVideo> meizhiWithVideos) {        
                return Observable.from(meizhiWithVideos);    
          }
    })
    .toSortedList(new Func2<MeizhiWithVideo, MeizhiWithVideo, Integer>() {    
          @Override    
          public Integer call(MeizhiWithVideo meizhiWithVideo1, MeizhiWithVideo meizhiWithVideo2) {        
                return meizhiWithVideo2.publishedAt.compareTo(meizhiWithVideo1.publishedAt);    
          }
    })
    .subscribeOn(Schedulers.computation());

#####iv. 排序后，我们得到Observable<List<MeizhiWithVideo>>数据源，传给adapter去更新UI
上面的`toSortedList(xxx)`方法会把`Observable<MeizhiWithVideo>`排序后重新组装成`Observable<List<MeizhiWithVideo>>`对象`sortedMVListObservable`，该对象对外发射一个`有序的List<MeizhiWithVideo>`。我们将该数据源提供给adapter供显示。

代码如下：

    sortedMVListObservable.observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Subscriber<List<MeizhiWithVideo>>() {    
        @Override    
        public void onCompleted() {            
            setRefresh(false); // stop refreshing data.                 
        }    
        @Override    
        public void onError(Throwable e) {    

        }
        @Override    
        public void onNext(List<MeizhiWithVideo> meizhiWithVideoList) {    
            adapter.setData(meizhiWithVideoList);
            adapter.notifyDataSetChanged(); // update UI
        }
    })

###4. 利用`Subscription`来管理异步处理与Activity生命周期
对于异步我们知道一直存在一个问题，假设一个页面要同时发出很多个http请求，如http1, http2, http3...，然后这些请求会被放在一个队列里依次发出，而且每个请求发出后需要等待一段时间才能得到返回数据。

那么问题就来了，假设在A页面发出了多个网络请求，在这些网络请求还在等待响应时用户就跳转到了B页面，在以前的情况下是，A页面的网络请求仍然进行直到所有数据返回，而且当数据返回时会尝试去调用A页面的UI进行修改，而此时已经进入了B页面，所以，这不仅造成了网络资源的浪费，也存在一定的风险。

有了RxJava，我们可以把每一个网络请求转化为一个`Subscription`对象，这个`Subscription`对象可以被手动`unsubscribe`，即停止订阅所请求的数据源，这样就可以暂定数据请求，而且即使数据返回回来，由于我已经取消订阅了，所以不会再接收到这些数据了。

代码实现：
在`BaseActivity`中，创建一个`CompositeSubscription`对象来进行管理

    `BaseActivity`
    private CompositeSubscription mCompositeSubscription;
    protected void addSubscription(Subscription s) {   
       if (this.mCompositeSubscription == null) {                
            this.mCompositeSubscription = new CompositeSubscription();    
        }    
        this.mCompositeSubscription.add(s);
    }

    @Override 
    protected void onDestroy() {    
          super.onDestroy();    
          if (this.mCompositeSubscription != null) {                
                this.mCompositeSubscription.unsubscribe();    
          }
    }

在实际的Activity中的网络请求:

    public class MyActivity extends BaseActivity {
      
        private void loadData() {
            Subscription s = gankApi.getMeizhiList(10)                           
                                         .subscribeOn(Schedulers.io())
                                         .observeOn(AndroidSchedulers.mainThread())
                                         .subscribe(...);
            addSubscription(s);
        }
    }

##三、改进及总结
本文通过对开源项目[Meizhi Android](https://github.com/drakeet/Meizhi)进行分析，了解了`Retrofit`，`RxJava`的实际应用场景，也对于二者有了更加深入的认识。

不过本人认为该项目还有一些可以改善的地方，比如`Retrofit`中利用`DrakeetFactory`工厂来生成`GankApi`的单例，但是`new DrakeetRetrofit().getGankApi();`也是一个可以生成`GankApi`的方法，而且是`public`的，那么如果新的开发者忘记调用`DrakeetFactory`来生成`GankApi`的实例，而是采用后者，那么工厂模式就达不到预期的目的了。我认为可以把`new DrakeetRetrofit().getGankApi();`这个操作内容放在`DrakeetFactory`工厂内部，并且设置为`private`属性，这样的话如果想要获得`GankApi`实例，就必须依靠`DrakeetFactory`来生成，从而真正保证了`单例`的优势。

最后，如果读者有意见欢迎评论，本人后续还会挑选优质的开源项目，分析其精髓，供读者学习领悟。

谢谢！

wingjay


### 关于作者wingjay

欢迎各位关注
[我的Github](https://github.com/wingjay): [https://github.com/wingjay](https://github.com/wingjay)
和
[我的个人博客](http://wingjay.com/): [http://wingjay.com](http://wingjay.com/)
和
[微博 iam_wingjay](http://weibo.com/u/1625892654): [http://weibo.com/u/1625892654](http://weibo.com/u/1625892654)

如果有问题，可以给我留言或发邮件[yinjiesh@126.com](mailto:yinjiesh@126.com)
![wingjay](https://avatars0.githubusercontent.com/u/9619875?v=3&s=460)
