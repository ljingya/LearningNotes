相信通过之前的入门章节，开发者已经对 React Native 有了基本的了解和使用，在这一章节会通过实战项目开发一个完整的 app,主要功能包含启动页，登陆页，注册页，首页，个人中心页，书单详情页，侧滑页。

###### 5.1.1 创建项目

在开发项目之前需要选择一款开发工具，在这里推荐使用 Visual Studio Code 作为开发 Reat Native 的工具，也可以选择使其他的开发工具。

React Native项目主要通过命令行来创建一个项目：

1. 创建项目

    在这次实战项目中是基于 React Native 0.55 版本,在某个文件夹下打开命令行，调用 **react-native init ShuDanApp --version 0.55.0**。当指定使用ReactNative的某个版本的时候必须精确到两个小数点。

2. 运行项目
   
    进入项目根目录，在 iOS 中调用 **react-native run-ios** 命令运行项目，在 Android 中使用 **react-native run-android** 命令运行项目。

到这里，我们的实战项目就创建完成了。

###### 5.1.2 项目结构介绍
​	React Native 项目主要由 Android 工程，iOS 工程，及 React Native 的 js 部分,项目结构如下:

```
│  App.js
│  index.js
│  package.json
│  
├─android
│          
├─app
│  ├─common
│  │      
│  ├─component
│  │      
│  ├─img
│  │      
│  ├─navigate
│  │      
│  ├─util
│  │      
│  └─view
│      ├─account
│      │      
│      ├─drawer
│      │      
│      ├─login
│      │      
│      └─shudan
│              
└─ios	
```

1. index.js
    ```
    import { AppRegistry } from 'react-native';
    import App from './App';
    AppRegistry.registerComponent('ShuDanApp', () => App);
    ```

    该文件是整个程序的入口,在这个文件中会将 App.js 创建的组件注册进来。

 2. App.js

    ```
    export default class App extends Component {
    	...
    	render() {
    	return (
      	<RootStack />
          	 		);
      	 	}	
    	}
    ```

    该文件是整个程序的初始组件，这个组件会在index.js中进行注册。

3. android目录

    在该文件夹下会生成一个 Android 项目工程。

4. ios目录

    在该文件夹下会生成一个 iOS 项目的工程。

5. app文件夹

    为了将项目的 React Native 页面放置在一个目录中，在项目的根目录新建了一个名称为 app 的文件夹作为存放 React Native 页面的目录。在这个目录结构下根据不同的业务模块对目录进行了分层。具体的分层:

    层级目录 | 功能
    ---|---
    common | 公用的样式文件和公用的常量文件等
    component | 抽取的功能组件
    img| 项目中使用的图片资源
    navigate目录| 对路由统一管理的目录
    util| 工具文件
    view| 页面文件的目录，在这个目录下会根据具体提的业务划分不同的文件夹

6. package.json

    这个文件下的内容主要是 npm 需要执行的脚本，和依赖包的名称和版本号。   

    ```
    {
        "name": "ShuDanApp",
        "version": "0.0.1",
        "private": true,
        "scripts": {
        "start": "node node_modules/react-native/local-cli/cli.js start",
        "test": "jest",
        "bundle-ios":"node node_modules/react-native/local-cli/cli.js bundle --entry-file 	index.js --platform ios --dev false --bundle-output ./ios/bundle/index.ios.jsbundle --assets-dest ./ios/bundle"
        },
        "dependencies": {
        "react": "16.3.0-alpha.0",
        "react-native": "0.55.0",
        "react-native-scrollable-tab-view": "^0.8.0",
        "react-navigation": "^2.11.2"
        },
        "devDependencies": {
        "babel-jest": "23.4.2",
        "babel-preset-react-native": "4.0.0",
        "jest": "23.5.0",
        "react-test-renderer": "16.3.0-alpha.0"
        },
        "jest": {
        "preset": "react-native"
            }
        }
    ```

    在 view 的目录下创建一个名叫 login 的文件夹，并在这个目录下创建三个文件 LoginView.js  (登陆组件), SplashView.js  (启动组件)，RegisterView.js  (注册组件)。这一部分功能需要实现启动页面组件，登陆页面组件，注册页面组件以及页面间跳转的路由。

- 路由

    路由的实现需要用到 React Navigation，因此在使用前需要调用命令 yarn add react-navigation 安装该包。这样就可以使用该包 下的导航组件了。
    
    首先为了统一管理页面组件，在 navigte 目录下新建 AppStack.js，在该文件下创建路由组件，将每个需要跳转的页面注册到该路由中，由于 app 启动的第一个页面是启动组件，因此将路由的 initialRouteName 属性设置为启动页面，代码如下：
    
    ```
    export const StackNavigator = createStackNavigator(
    {
    Splash: {
      screen: SplashView,
      navigationOptions: {
        header: null
      }
    },
    Login: {
      screen: LoginView,
      navigationOptions: {
        header: null
      }
    },
    Register: {
      screen: RegisterView,
      navigationOptions: {
        title: '注册'
      }
    },
    ShuDanDetail: {
      screen: ShuDanDetailView,
      navigationOptions: {
        title: '书单详情'
      }
    },
    Tab: {
      screen: TabBottomBar,
      navigationOptions: {
        header: null
            }
        }
    },
    {
    initialRouteName: 'Splash',
    navigationOptions: {
      headerStyle: {
        backgroundColor: '#353535',
        height: 44
      },
      headerTintColor: '#FFFFFF',
      headerTitleStyle: {
        fontWeight: 'bold',
        flex: 1,
        textAlign: 'center',
        fontSize: 18
      },
      headerRight: <View />
            }
        }
    );
    ```
     然后将路由组件 RootStack 需要添加到 App.js 中的 App 组件里面。代码如下：
    ```
    export default class App extends Component {
    constructor(props) {
    super(props);

    }
    render() {
    return (
      <RootStack />
            );
        }
    }
    ```
    
    这样当应用启动的时候，应用呈现的第一个页面便是启动组件的内容。
- 启动组件
  
    该组件需要实现以下功能，设置一张启动页面的图片，及设置一秒延时跳转到下一个页面，当应用第一次启动需要跳转到登陆页面，登陆完成之后当下一次启动直接跳转到首页。
    
    布局主要由一个根 View 组件嵌套一个 Image 组件。代码如下：
     ```
    <View style={styles.root}>
        <Image source={require('../../img/default.png')} />
    </View>
     ```
    
     跳转逻辑主要实现，在 componentDidMount 方法中设置1秒延时的定时器，在跳转页面前先从 AsyncStorageUtil 中获取userName，AsyncStorageUtil 是对 AsyncStorage 这个异步存储的 api 的简单封装。若获取到这个存储的值，说明不是第一次登陆，跳转到首页，若为空，说明是第一登陆或者存储的值当退出登陆的时候被清除了，需要跳转到登陆页面。另外当组件销毁的时候需要清除定时器，代码如下:
    ```
     /**
     * 设置定时器
     */
    componentDidMount() {
        this.timer = setTimeout(() => {
            AsyncStorageUtil.getValue("userName", (error, result) => {
                if (result) {
                   this.props.navigation.navigate('Tab');
                }else{
                    this.props.navigation.navigate('Login');
                }
            });

        }, 1000);
    }
    /**
     * 清除定时器
     */
    componentWillUnmount() {
        this.timer && clearTimeout(this.timer);
    }
    ```
    
- 登陆组件
  
    登陆组件主要实现账号输入框宫功能及密码输入框功能，登陆功能和跳转注册组件的功能。

    最终页面效果如下:
    
    ![图5-5](https://note.youdao.com/yws/api/personal/file/A1124736634040859E60E6912D4BC3F7?method=download&shareKey=8b5ee5636bf2d9daec0076dd7e2a0048)
    

   布局代码主要用 TextInput 组件及 TouchableOpacity 组件实现,代码如下:
​    

```
<View style={[CommonStyle.root, { backgroundColor: '#FFFFFF' }]}>
        <Text style={styles.text_header}>您好</Text>
        <Text style={styles.text}>欢迎来到登陆界面</Text>
        <TextInput
            style={styles.input}
            numberOfLines={1}
            placeholder={'请输入账号'}
            placeholderTextColor={'#828181'}
            underlineColorAndroid="transparent"
            onChangeText={this.onChangeTextUserName}
            value={this.state.userName}
        />
        <TextInput
            style={styles.input}
            numberOfLines={1}
            placeholder={'请输入密码'}
            placeholderTextColor={'#828181'}
            underlineColorAndroid="transparent"
            onChangeText={this.onChangeTextPsw}
            value={this.state.psw}
            />
        <Text style={styles.text_register_desc}
            onPress={() => this.onRegisterPress()}
                >
            没有账号？注册一个吧
        </Text>
        <TouchableOpacity
            onPress={() => this.onPress()}
            style={{ marginTop: 26, alignItems: 'center' }}>
            <Image source={require('../../img/login_btn.png')} />
       </TouchableOpacity>
    </View>
```

​      

账号及密码输入框功能：

 由于输入账号框和密码框的 TextInput 组件的 value 属性的值是可变化的，因此将账号和密码的初始值设置在 state 中.。
 ```
 constructor(props) {
    super(props);
    this.state = {
        userName: "",
        psw: ""
    }
}
 ```
 当输入框的内容变化时分别调用 onChangeTextUserName 和 onChangeTextPsw 方法，并将变化的值，通过 setState 方法重新设置给 state 中的 userName 和 psw 字段，会触发Render方法重新渲染。

onChangeTextUserName 方法：

```
/**
 * 账号输入
 */
onChangeTextUserName = (text) => {
  this.setState({
      userName:text
  })
}
```
onChangeTextPsw 方法：
```
 /**
* 密码输入
*/
onChangeTextPsw = (text) => {
    this.setState({
        psw:text
    })
}
```
另外如果app不是第一登陆需要先获取之前存储的 userName 并设置给 state 中的 userName 属性，实现如下代码:
```
 componentDidMount() {
    AsyncStorageUtil.getValue("userName", (error, result) => {
        if (result) {
            this.setState({
                userName: result
            })
        }
    });
}
```
登陆功能：

当点击登陆按钮时，校验非空，然后将正确的值存储下来，调用路由的跳转方法跳转到下一个页面。
```
  /**
 * 按钮跳转
 */
onPress = () => {
    if (isEmpty(this.state.userName)) {
        toastShort("账号不能为空");
        return;
    }
    if (isEmpty(this.state.psw)) {
        toastShort("密码不能为空");
        return;
    }
    let multiParis = [
        ["userName", this.state.userName],
        ["psw", this.state.psw]
    ]
    AsyncStorageUtil.setValues(multiParis, (errors) => {
        this.props.navigation.navigate('MainDrawer')
    }).then(() => this.props.navigation.navigate('MainDrawer'))

}
```


​    
- 注册页面
  
    注册页面主要由，注册账号的输入框和注册密码的输入框功能，注册功能。

    最终页面效果如下:

    ![图5-6](https://note.youdao.com/yws/api/personal/file/F29AF56D62254C809B8FC768BF122B6C?method=download&shareKey=9b532e5974e0d46c05ff569094954720)
  
    布局主要由两个 TextInput 组件及 TouchableOpacity 组件实现，代码如下：
    ```
      <View style={[CommonStyle.root, { backgroundColor: '#FFFFFF' }]}>
                <View
                    style={{ alignItems: 'center', marginBottom: 45 }}>
                    <Image
                        source={require('../../img/register_header.png')}
                        style={{ marginTop: 45, marginBottom: 45 }} />
                    <Text
                        style={{ color: '#636362', fontSize: 18 }}>
                        您好
                    </Text>
                    <Text
                        style={{ color: '#636362', fontSize: 14, marginTop: 10 }}>
                        欢迎来到注册页面
                    </Text>
                </View>

                <TextInput
                    style={styles.input}
                    numberOfLines={1}
                    placeholder={'请输入账号'}
                    placeholderTextColor={'#828181'}
                    underlineColorAndroid="transparent"
                    onChangeText={this.onChangeTextUserName}
                    value={this.state.userName}
                />

                <TextInput
                    style={styles.input}
                    numberOfLines={1}
                    placeholder={'请输入密码'}
                    placeholderTextColor={'#828181'}
                    underlineColorAndroid="transparent"
                    onChangeText={this.onChangeTextPsw}
                    value={this.state.psw}
                />

                <TouchableOpacity
                    onPress={() => this.onRegisterPress()}
                    style={{ marginTop: 26, alignItems: 'center' }}>

                    <Image
                        source={require('../../img/register_btn.png')} />

                </TouchableOpacity>
            </View>
    ```
    注册账号的输入框与注册密码的输入框功能:
    
    当输入框的内容变化时分别调用 onChangeTextUserName 和 onChangeTextPsw 方法，并将变化的值，通过 setState 方法重新设置给 state 中的 userName 和 psw 属性，接着触发 Render 方法重新渲染。
    ```
    /**
    * 账号输入
    */
    onChangeTextUserName = (text) => {
        this.setState({
            userName: text
        })
    }
    /**
    * 密码输入
    */
    onChangeTextPsw = (text) => {
        this.setState({
            psw: text
        })
    }
    ```
    注册功能:
    
    当点击注册按钮时，校验注册账号密码非空，然后回退到登陆页面。
    ```
    /**
     * 注册
     */
    onRegisterPress = () => {
        if (isEmpty(this.state.userName)) {
            toastShort("账号不能为空");
            return;
        }
        if (isEmpty(this.state.psw)) {
            toastShort("密码不能为空");
            return;
        }
        toastShort("注册成功");
        this.props.navigation.goback();
    }
    ```
    

首页包含了底部的 TabBar 和书单列表页面两部分。底部的 TabBar 的实现使用 React Navigation 的 createBottomTabNavigator 实现。书单 ShuDanView 组件，列表头部的标签使用 react-native-scrollable-tab-view 这个库实现，对于该库具体的使用可以在github上搜索该库，查找详细的使用方法，这里只介绍用到属性和方法。另外为了将标签的逻辑和列表的逻辑抽离，在view目录下的书单目录中创建 ShuDanListView.js 文件，将列表功能的逻辑抽取到 ShuDanListView 组件中。

效果图如下:

![图5-7](https://note.youdao.com/yws/api/personal/file/1DC05175A02A498F9D4310A5BF283BC3?method=download&shareKey=b3f69e4d650150be78bd7d4318fcfed9)

- 底部组件（TabBottomBar）
  
   在 AppStack.js 中创建底部导航,创建导航使用上述提到的 createBottomTabNavigator，将底部的书单页面和我的页面注册到该底部导航中，然后设置各自的 navigationOptions 属性例如图标，文字等。然后将 TabBottomBar 组件像登陆和启动页面一样需要注册到RootStack这个路由组件中。代码如下:
    ```
    export const TabBottomBar = createBottomTabNavigator({
    ShuDan: {
    screen: ShuDanView,
    navigationOptions: {
      tabBarPosition: 'bottom',
      tabBarLabel: '首页',

      tabBarIcon: ({ focused, tintColor }) => {
        let imgSource = focused ? require('../img/icon2.png') : require('../img/home_unselect.png');
        return <Image
          style={{ width: 25, height: 25 }} source={imgSource} />;
      }
    }
    },
    Me: {
    screen: MeView,
    navigationOptions: {
      tabBarPosition: 'bottom',
      tabBarLabel: '我的',

      tabBarIcon: ({ focused, tintColor }) => {
        let imgSource = focused ? require('../img/me_selected.png') : require('../img/icon3.png');
        return <Image style={{ width: 25, height: 25 }} source={imgSource} />;
            },
        }
    }
    },
    {
    tabBarOptions: {
      activeTintColor: ' #636362',
    }
     }
    );
    ```


- 书单组件（ShuDanView）
  
    在 view 目录下创建 shuadan 目录，这个目录存放整个书单业务的全部组件。在书单目录下创建 ShuDanView.js 文件，并创建 ShuDanView 组件。书单顶部的tab标签使用的是 react-native-scrollable-tab-view 组件完成，在使用前也需要先安装该库，使用 yarn add react-native-scrollable-tab-view  安装该包，然后导出该包下的两个组件，
    ```
    import ScrollableTabView, {  ScrollableTabBar } from 'react-native-scrollable-tab-view';
    ```
    ScrollableTabView：用来嵌套列表组件。需要在嵌套的子组件中添加tabLabel属性，表示子组件对应标签的名称。
    
    ScrollableTabBar：一个可滑动的tab标签组件。
    
    该页面的具体布局，主要由如下ScrollableTabView组件和的 ShanDanListView 列表组件组成，在该布局中调用 renderList 方法，返回列表组件，另外为了在书单组件中处理列表组件中的点击事件和获取到需要传递到下一个页面的值，因此为 ShanDanListView 组件添加了一个 onPress的属性值，用于处理点击事件。代码如下:
    ```
    render() {
        return (

            <ScrollableTabView
                renderTabBar={() => <ScrollableTabBar />}
                style={{ flex: 1, backgroundColor: '#FFFFFF' }}
                tabBarBackgroundColor="#FFFFFF"
            >
                {
                    this.renderList()
                }
            </ScrollableTabView>

        );
    }
    
     /**
       * 返回列表组件
       */
    renderList = () => {
        return (
            this.state.shuDanTabsArray.map((value, index) => {
                return <ShanDanListView
                    tabLabel={value.name}
                    onPress={(name) => this.onPress(name)}
                />
            })
        )
    }
      
     /**
      * 点击列表Item的回调并调用路由的跳转方法，第二个参数是传递到下一个页面的参数
      */
    onPress = (name) => {
        this.props.navigation.navigate('ShuDanDetail', { name: name });
    }
    ```
    
    请求网络功能的代码如下：
    ```
    
    componentDidMount = () => {
        this.getTabs();
    }

    getTabs = () => {
        let url = Constant.baseUrl + 'action=category';
        let _this=this;
        let callBack = {
            onSuccess(resultObject) {
                _this.setState({
                    shuDanTabsArray: _this.state.shuDanTabsArray.concat(resultObject)
                });
            },
            onError(code, errorMsg) {

            }
        };
        httpFetch(url, 'GET', null, callBack);
    }
    ```
    当组件挂载的时候请求网络，返回成功的时候将返回的数据设置给 shuDanTabsArray 该数组。当调用 this.setState 方法之后，会重新渲染该界面。
    
- 书单列表组件（ShanDanListView）

    ShanDanListView 组件主要使用 FlatList 组件， 由于是网格布局，将其numColumns属性设置为2，在data属性中设置state值，在 renderItem 设置返回布局的每个 Item 组件。当点击 Item 组件 的时候会触发 TouchableOpacity 的onPress 方法。在 onPress 方法中调用 this.props.onPress(name)，触发书单组件中的 onPress 方法，布局代码如下：
    ```
    <FlatList
        data={this.state.shuDanItemArray}
        renderItem={this.renderItem}
        numColumns={3}
        columnWrapperStyle={{justifyContent:'space-around'}}
        />
            
      
    /**
      * 返回每个Item的布局
      */
    renderItem = ({ item }) => {
        let imageSource = require('../../img/default.png');
        if (item.image) {
            imageSource = { uri: item.image };
        }
        return (<TouchableOpacity style={[styles.item_root]} onPress={() => this.onPress(item.id)}>

            <Image style={styles.img}source={imageSource} resizeMode='center'/>
            <Text style={styles.text}>{item.name}</Text>
        </TouchableOpacity>)
    }
    
    /**
    * item的点击事件，在该方法中调用书单组件的onPress()方法
    */
    onPress = (name) => {
        this.props.onPress(name);
    }
    ```
    请求列表数据的网络请求如下：
    ```
     componentDidMount = () => {
       this.getList();
    }

    getList=()=>{
        let url = Constant.baseUrl + 'action=list&q='+this.props.tabLabel;
        let _this=this;
        let callBack = {
            onSuccess(resultObject) {
                _this.setState({
                    shuDanItemArray: _this.state.shuDanItemArray.concat(resultObject)
                });
            },
            onError(code, errorMsg) {

            }
        };
        httpFetch(url, 'GET', null, callBack);
    }
    ```
    在组件挂载的时候请求网络，在获取成功数据之后，通过 setState 方法设置 shuDanItemArray 数组，然后重新渲染该组件。

###### 5.2.3 个人中心页面
个人中心页面主要显示个人的信息功能，因此这个页面主要是显示功能，侧重布局功能，无其他业务逻辑。

效果图:
​    
![图 5-9](https://note.youdao.com/yws/api/personal/file/6F1E907FFAA64B728D852F2150E68339?method=download&shareKey=f307871146d08a7aed9eddb7125321f0)


布局使用图片做背景时，使用了 ImageBackground 组件，代码如下:
```
 <View style={[CommonStyle.root]}>
    <View style={styles.header_layout}>
        <Image source={require("../../img/me_header.png")} />
        <Text style={styles.header_text}>我的</Text>
        <Image source={require("../../img/me_setting.png")} />
    </View>
                
    <View style={styles.header_view}>
        <Text style={{ marginTop: 24, fontSize: 18, color: '#353535' }}>
            Jack
        </Text>
        <Image source={require('../../img/default_icon.png')}
                        style={styles.header_image} />
        <ImageBackground
                        resizeMode='contain'
                        source={require('../../img/me_rect.png')}
                        style={styles.imageBackground}>
            <View>
                <Text style={{ fontSize: 17, color: '#FFFFFF' }}>强力推荐卡</Text>
                <Text style={{ fontSize: 14, color: '#FFFFFF', marginTop: 12 }}>
                    最给力的书单就在这里
                </Text>
            </View>
            <Image source={require('../../img/me_lingqu.png')} />
        </ImageBackground>
    </View>

    <View style={styles.like_book_view}>
        <Image source={require('../../img/me_like.png')} />
        <Text style={{ flex: 1, marginLeft: 8, color: '#636362', fontSize: 14 }}>
            我想要的书籍
        </Text>
        <Text style={{ color: '#636362', fontSize: 21 }}>
            0<Text style={{ fontSize: 14 }}>本</Text>
        </Text>
    </View>
                
    <View style={styles.shoucang_book}>
        <Image source={require('../../img/me_shoucang.png')} />
        <Text style={{ flex: 1, marginLeft: 8, color: '#636362', fontSize: 14 }}>
            我收藏的书籍
        </Text>
        <Text style={{ color: '#636362', fontSize: 21 }}>
            0<Text style={{ fontSize: 14 }}>本</Text>
        </Text>
    </View>
                
    <View style={styles.dianzan_book}>
        <Image source={require('../../img/me_dianzan.png')} />
        <Text style={{ flex: 1, marginLeft: 8, color: '#636362', fontSize: 14 }}>
            我点赞的书籍
        </Text>
        <Text style={{ color: '#636362', fontSize: 21 }}>
            0<Text style={{ fontSize: 14 }}>本</Text>
        </Text>
    </View>
</View>
```
###### 5.2.4 书单详情
 shudan 目录下创建 ShuDanDetailView.js ，ShuDanDetailView 组件主要包括书籍评论列表功能，发表评论的功能，这个界面主要采用 ScrollView 和 FlatList 组件实现。

效果图如下:

![图 5-10](https://note.youdao.com/yws/api/personal/file/F8EA7B92888C407794AFCA28B8EA7337?method=download&shareKey=364a69ca81acaa4c6209c564dc033a7f)

布局实现是 ScrollView 嵌套 FlatList，底部是评论框部分代码如下:
```
<View style={[CommonStyle.root]}>
    <ScrollView >
        <View style={styles.header_view}>
            <Image 
                style={styles.img}
                source={this.getImage()} 
                resizeMode='contain'
                />
            <Text style={styles.header_text}>
                {'       ' + this.state.introduce}
            </Text>

            <Image resizeMode={'stretch'}
                source={require('../../img/shudan_underline.png')}
                style={styles.img_underline} />
                <View style={{ flexDirection: 'row-reverse', paddingBottom: 23 }}>
                <Image source={require('../../img/shudan_shoucang.png')}
                style={{ marginRight: 11 }} />
                <Text style={styles.text_shoucang}>收藏</Text>
                </View>
        </View>

        <View style={styles.view_liuyan}>
            <Text style={styles.text_liuyan}>
                全部留言
            <Text style={{ fontSize: 12 }}>
                (共{this.state.commentArray.length}条)
            </Text>
            </Text>
            <Image resizeMode={'stretch'}
                source={require('../../img/shudan_underline.png')}
                style={{ height: 1, width: Constant.screenWidth - 34 }} />
        </View>
        <FlatList
            data={this.state.commentArray}
            renderItem={this.renderItem}
            />
    </ScrollView>
    <View style={styles.view_input}>
        <TextInput
            style={styles.input}
            placeholder={'请输入评论'}
            placeholderTextColor={'#828181'}
            underlineColorAndroid="transparent"
            maxLength={50}
            onChangeText={this.onChangeText}
            value={this.state.text}
            />
        <View style={styles.view_btn}>
            <Button title={'发送'}
                    onPress={this.onPress}
                    color="#4698CA" />
        </View>
    </View>
</View>
```

###### 构造函数:
在构造函数中，初始化了十条评论数据，并将该页面需要变化的值设置到 state 中，

commentArray:评论列表的值

text:输入框的值

image:图片的地址

introduce：书籍的介绍文字的值


```
constructor(props) {
    super(props);
    let commentArray = [];
    for (let i = 0; i < 10; i++) {
        commentArray.push({ name: 'tom', content: '好书好书' })
        }
    this.state = {
        commentArray: commentArray,
        text: "",
        image: "",
        introduce: ""
    }
}
```

网络请求功能:

在组件挂载的时候去请求数据，另外在 getDetail 方法中需要获取上一个页面设置在 navigation 中的值，通过  navigation 的 getParam 方法获取。在请求成功之后调用 setState 方法重新渲染页面。

```
componentDidMount() {
    this.getDetail();
}

/**
 * 获取详情
 */    
getDetail = () => {
    const { navigation } = this.props;
    let name = navigation.getParam('name', null);
    if (!name) {
        return;
        }
    let url = Constant.baseUrl + 'action=detail&q=' + name;
    let _this = this;
    let callBack = {
        onSuccess(resultObject) {
            _this.setState({
                image: resultObject.data.image,
                introduce: resultObject.data.introduce
                });
            },
        onError(code, errorMsg) {

            }
        };
    httpFetch(url, 'GET', null, callBack);
}
```
这样基本就实现了详情的头部跟评论列表部分。接下来实现如何发送评论并更新列表。


###### 发表评论

当输入框内容变化时会调用 onChangeText 方法，在该方法中调用 setState 方法设置 text 的值。

```
/**
* 输入框内容变化
*/
onChangeText = (str) => {
    this.setState({
    text: str
    })

}
```
点击发送按钮的时候调用 onPress 方法,首先判断了 text 的值是否为空，若为空直接返回，不为空和组装了下数据对象并调用 setState 方法重新渲染页面。

```
/**
* 点击发送
*/
onPress = () => {
if (isEmpty(this.state.text)) {
    toastShort("不能为空", false);
        return;
    }
    this.setState({
    commentArray: this.state.commentArray.concat({ name: 'tom', content: this.state.text }),
            text: ""
        })
}
```

###### 5.2.5 侧滑页面


侧滑页面使用 React Navigation中 的 createDrawerNavigator 创建，因此在 AppStack.js 中创建一个侧滑组件。另外需要将侧滑组件和 TabBottomBar 组件结合起来。

效果图：

![图5-11](https://note.youdao.com/yws/api/personal/file/3DCB19DBE4754841B4CFB670ADD8AE71?method=download&shareKey=4c434d5158fd1622447e0d4b3568d820)

创建侧滑组件:

在首页中的页面中，使用 createBottomTabNavigator 创建了 TabBottomBar 组件，然后将 TabBottomBar 组件注册到了 StackNavigator 中,实现侧滑，需要在 StackNavigator 外再使用一层 DrawerNavigator 。代码如下。
```
export const StackNavigator = createStackNavigator(
  {
    Splash: SplashView,
    Login: LoginView,
    Register: RegisterView,
    ShuDanDetail: ShuDanDetailView,
    Tab:  TabBottomBar  
  },
  {
    initialRouteName: 'Splash',
    navigationOptions: {
      headerStyle: {
        backgroundColor: '#353535',
        height: 44
      },
      headerTintColor: '#FFFFFF',
      headerTitleStyle: {
        fontWeight: 'bold',
        flex: 1,
        textAlign:'center'
      }
    }
  }
);
export const RootStack = createDrawerNavigator({
  Tab: {
    screen: StackNavigator,
    navigationOptions :{
      drawerLabel: <DrawerView/>
    }
  }
})
```

使用 createDrawerNavigator 创建侧滑组件，将之前使用 StackNavigator 创建的组件添加进去，另外需要设置侧滑组件的页面，使用 navigationOptions 下的 drawerLabel 属性，它的值可以是一个字符串或者一个组件。在view目录下的 drawer 目录中 DrawerView.js 中创建 DrawerView 组件，导入到 AppStack.js 中。
```
export const RootStack = createDrawerNavigator({
  Tab: {
    screen: StackNavigator,
    navigationOptions :{
      drawerLabel: <DrawerView/>
    }
  }
})
```
DrawerView组件：

布局代码如下:
```
<View style={[CommonStyle.root, { backgroundColor: '#FFFFFF' }]}>
    <View style={{ alignItems: 'center' }}>
                
        <ImageBackground
            source={require('../../img/drawer_icon2.png')}
            style={styles.imgbackground}>
            <View style={styles.top_view}>
                <Image source={require('../../img/default_icon.png')} />
                <Text style={styles.text_name}> Jack</Text>
            </View>
            <Text style={styles.text_desc}>送给程序员的爱心书单</Text>
        </ImageBackground>

        <Image
            source={require('../../img/drawer_icon1.png')}
            style={styles.img} />
    </View>

    <View style={styles.bottom_view}>
        <Image source={require('../../img/drawer_icon3.png')} />
        <Image source={require('../../img/drawer_arrow.png')} />
    </View>
</View>
```

###### 5.3.1 安卓打包
在发布应用的时候，需要生成一个带签名的 apk ，在安卓中生成签名有两种方式，一种利用 AndroidStudio 的可视化界面生成签名，另外一种使用jdk下的bin目录中 keytool 工具生成一个签名。本章只对命令行方式说明如何生成签名。
1. 调出命令行，可能需要进入到安装 jdk 目录下的 bin 目录，在命令行中输入如下命令:
    ```
    keytool -genkey -v -keystore D:\work_android\ShuDanApp\ShuDanApp\shudan.keystore -alias shudan-alias -keyalg RSA -keysize 2048 -validity 10000
    ```
    由于在 C 盘中有读写权限，因此在 keystore 后面指定生成签名的目录，然后调用此命令之后需要你输入密钥库的密码，姓名等信息，最后就生成一个 shudan.keystore 文件。
2. 生成签名之后需要在 React Native 项目中 android 工程中配置生成的签名信息。
    - 在 gradle.properties 文件中存放生成的签名信息。

        STORE_FILE:签名文件存放的目录

        KEY_ALIAS:命令行中 -alias 后面的参数也就是别名

        STORE_FILE_PASSWORD: 密钥库密码

        KEY_ALIAS_PASSWORD:别名密码
        ```
        STORE_FILE=../../shudan.keystore
        KEY_ALIAS=shudan-alias
        STORE_FILE_PASSWORD=123456
        KEY_ALIAS_PASSWORD=123456
        ```

    - 在 android/app/build.gradle 中配置签名，在 android 域下添加 signingConfigs 配置，并且在 buildTypes 中的 release 域中配置 signingConfig 。然后输入 release 包时就使用 signingConfigs 下 release 配置的签名信息。

        ```
        android {
        ....
        signingConfigs {
        release {
                storeFile file(STORE_FILE)
                storePassword STORE_FILE_PASSWORD
                keyAlias KEY_ALIAS
                keyPassword KEY_ALIAS_PASSWORD
        }
        }
        
        buildTypes {
                release {
                ...
               signingConfig signingConfigs.release
                    }
               }
        
        }
        ```

3.生成apk

在终端的项目根目录下输入 cd android，进入 android 目录中，再调用 ./gradlew assembleRelease 命令，在 macOs,Linux 或是 windows 下的powerShell环境中，./ 不可以省略，在 windows 下的 cmd 命令行中需要去掉。最终会在 android/app/build/outputs/apk 目录下生成 app-release.apk 文件，这个文件就是最终的签名文件。如下图:

![Image](https://note.youdao.com/yws/api/personal/file/4D068CC3B34045A58BBF8470445041FC?method=download&shareKey=6d3c1b300329a6b268956618d8125f76)

###### iOS 打包
1. 生成 bundle 文件

    在项目的 ios 目录下新建 bundle 目录，然后将打包命令配置在 package.json 中，在 scripts 域下添加 budle-ios 命令，然后在项目根目录运行npm run bundle-ios,会在 ios 目录下 bundle 目录下生成离线资源。配置如下:
    ```
    "scripts": {
    "start": "node node_modules/react-native/local-cli/cli.js start",
    "test": "jest",
    "bundle-ios":"node node_modules/react-native/local-cli/cli.js bundle --entry-file index.js --platform ios --dev false --bundle-output ./ios/bundle/index.ios.jsbundle --assets-dest ./ios/bundle"
     },
    ```
    --entry-file：React Native 项目的入口文件名称如 index.js。
    
    --platform：平台名称
    
    --dev：设置 false 时会对 js 代码进行优化
    
    --bundle-output：生成 jsbundle 文件的名称
    
    --assets-dest：图片及其他资源存放的目录

2. 添加到 Xcode 中

    在 Xcode 中使用 Create folder references方式添加 bundle 文件夹。
3. 修改 AppDelegate.m 文件

    在该文件下新加一段打包使用离线资源的代码如下：
    ```
    模拟器调试打开此行
    // jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil];
    打包开启此行代码
    jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"index" withExtension:@"jsbundle"];
    ```
    







