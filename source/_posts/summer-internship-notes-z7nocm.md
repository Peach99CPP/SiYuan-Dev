---
title: 暑期实习笔记
date: '2023-08-28 15:46:23'
updated: '2023-08-30 13:38:32'
permalink: /post/summer-internship-notes-z7nocm.html
comments: true
toc: true
---
# 暑期实习笔记

‍

## 说明

该实习笔记旨在记录我在暑期实习过程中所遇到的一些深入探索的内容。主要包含以下几点：

* 基于​ `QStackedWidget`​​ ​实现登录界面与主界面切换
* 如何实现对进程的捕获
* 数据库模块的使用
* HTML 中禁用对长按事件的响应
* 将 Qt 界面中的按键输入定向输出到 GBA 游戏界面中
* 子页面切换问题

---

## `QStackedWidget`​ 实现页面切换

在第一次使用页面切换功能时，使用的是 new 一个新的目标页面实例然后显示该实例的同时将当前页面进行隐藏的方法。这种方法简单易用，但是存在内存释放的问题，因为在进行页面切换的时候，会不停地申请内存，但是却没有使用 delete 方法释放已经不再使用的内存。

因此，为了解决这个问题，在经过 **资料查阅** 之后，决定采用 `QStackedWidget` ​来实现页面的管理操作。

思路如下：

1. 在初始化时就创建所需要切换显示的所有页面的实例对象
2. 将实例对象的以指针方式存储在 `QStackedWidget` ​的 stack 中，进行统一管理
3. 不具体显示某个页面，而是显示 `QStackedWidget` ​对象
4. 在需要进行界面切换时，通过调用 `QStackedWidget->setCurrentIndex` ​方法实现切换页面

通过这样子的方法，便无需在每次切换时候不断地 new 对象，也无需对应地进行 delete。

```c++
void Swicth_Page(bool same_gemeotry,int next_pageID,bool first_time)
{
    if(first_time)
    {
        page_stackPtr->setCurrentIndex(0);
        return;
    }
    //切换到下一个界面 并且读取当前界面的大小使得前后两个界面的大小、和位置一致
    int count = page_stackPtr->count();
    int  current_index = page_stackPtr->currentIndex();
    QWidget  *current_widget = page_stackPtr->widget(current_index);
    int next_index;
    if(next_pageID == -1) //default next one
    {
       next_index  = (current_index + 1) % count;
    }
    else{
        next_index = (next_pageID%count);
    }
    QWidget  *next_widget = page_stackPtr->widget(next_index);
    if(same_gemeotry)
    {
    //ensure the same gemotry
    next_widget->setGeometry(current_widget->geometry());
    }
    //切换到下一个界面
    page_stackPtr->setCurrentIndex(next_index);
}
```

---

## 进程捕获的实现

在实现GBA游戏机窗口嵌入Qt界面时，主要有以下步骤

1. 确定GBA游戏机模拟器进程的类名
2. 在模拟器运行时，对当前所有进程的类名进行正则匹配
3. 匹配到进程信息后，读取其winID
4. 根据winID将对应进程转换为QWindow对象
5. 将QWindow对象存储在QWidget中
6. 控制QWidget对象的显示

实现过程：

使用指令 `VisualBoyAdvance ​`​启动GBA模拟器  
从其窗口可知其大致类名。  
  

​![](https://peachpidgo.oss-cn-guangzhou.aliyuncs.com/20230828204104.png)​

使用 `xprop`​指令获取进程信息，分析出进程类名![根据信息分析进程类名](https://peachpidgo.oss-cn-guangzhou.aliyuncs.com/202308301338762.png "进程信息")​

根据所获取的进程信息可以得知：  
GBA模拟器进程的class name为： `VisualBoyAdvance'`​

根据所获知的类名，获取所有的进程类型并进行正则匹配，

```c++
//查找所有进程
QProcess *get_winId = new QProcess(this);
get_winId->setProcessChannelMode(QProcess::MergedChannels);
get_winId->start(queryCmd);
get_winId->waitForReadyRead();
QString wid = get_winId->readAll();
//进行正则匹配
QStringList list = wid.split("\n");
for (int i = 0; i < list.length(); i++)
{
    QString wmctrl_id = list[i];

    if (wmctrl_id.contains("Visual"))
    {
        qDebug() << wmctrl_id;
        Qwinid.append(wmctrl_id.left(10));
        int nHWindID = Qwinid.toUInt(Q_NULLPTR, 16);
        winid = (WId)(nHWindID);
    }
}
//对象转化
gameWindow = QWindow::fromWinId(winid);
gameWidget = QWidget::createWindowContainer(gameWindow, this);
ui->scrollArea->setWidget(gameWidget);
```

---

## 数据库模块的使用

本次数据库模块的学习 基于 **Sqlite3 ​**进行，主要学习了如何进行数据库的基本使用

* 表的增删改查
* 表中数据的增删改查
* 在Qt中利用 `sql`​ 模块进行数据库功能的实现

  * 屏保界面的图片来自于数据库中的数据
  * 数据库中存储每一款游戏的信息（名称、分类、描述、文件路径）
  * 对游戏信息的修改直接作用于数据库

其中，主要涉及到对表中数据的增删改查功能的实现。因为数据库中表的格式已经被预先定义好了，方便后续程序进行查找。以下是数据添加举例了如何使用数据库功能

```c++
    QSqlQuery queryAdd;
    queryAdd.prepare("INSERT INTO games (game_name, game_type, gba_path, game_image, game_description) \
    VALUES (:gameName, :gameType, :gbaPath, :gameImage, :gamedescription)");

    queryAdd.bindValue(":gameName", ui->game_name_le->text());
    queryAdd.bindValue(":gameType", ui->game_type_le->text());
    queryAdd.bindValue(":gbaPath", ui->game_file_le->text());
    queryAdd.bindValue(":gameImage", ui->game_image_le->text());
    queryAdd.bindValue(":gamedescription", ui->game_description_le->text());
```

---

## HTML中禁用长按响应

在使用HTTP服务器实现远程控制时，其界面由HTML进行描述实现。此时，由于远程控制界面中存在需要长按持续控制的使用场景，但是由于浏览器界面默认对于界面长按会触发二级菜单而不是预期中的长按会持续性输入控制信号。因此，需要在HTML中实现对界面长按动作的响应。

​![](https://peachpidgo.oss-cn-guangzhou.aliyuncs.com/202308301338215.png "HTML界面")​

经过查阅相关的文档以及网络信息之后，得到了禁用长按响应的方法：通过对`web-kit ​`​进行设置以实现对长按响应的禁用。同时为了考虑 **兼容性 ​**，需要对不同的浏览器内核都进行设置以保证其多平台的适用。

```html
<style> 
  * {
    -webkit-touch-callout: none;
    /*系统默认菜单被禁用*/
    -webkit-user-select: none;
    /*webkit浏览器*/
    -khtml-user-select: none;
    /*早起浏览器*/
    -moz-user-select: none;
    /*火狐浏览器*/
    -ms-user-select: none;
    /*IE浏览器*/
    user-select: none;
    /*用户是否能够选中文本*/
  }
</style>
//设置事件监听器
<script>
  document.addEventListener('contextmenu', event => event.preventDefault());
  document.addEventListener('selectstart', event => event.preventDefault());
</script>

```

---

## Qt输入重定向

将GBA模拟器的进程界面嵌入Qt窗口后，还需要在该界面设置对应的游戏控制按键以实现通过按键来玩游戏。因为在实际的开发场景中，游戏机并没有键鼠连接，无法通过键鼠进行控制。但在实验过程中发现存在问题：

在Qt窗口界面的按键，若对其进行操作，此时的光标焦点将聚焦在Qt界面中，而不是GBA模拟器界面中，如果此时直接按下按键的话，则按键输入的接收对象是Qt程序而不是GBA模拟器界面。这样子的话，便无法通过对Qt界面的按键进行操控来实现玩游戏了。

因此，需要解决的是将Qt界面中产生的输出 **重定向 ​**到GBA模拟器进程界面的输入，这样子的话才可以实现对游戏的有效输入控制。

经过资料的查阅后，发现可以通过 `xdotool`​模块实现该需求，通过调用该模块来实现将指定的按键行为输出到指定的进程窗口中

```c++
//向GBA模拟器进程输入Enter按键的
void GBA_Page::on_ctrlEnter_bt_pressed()
{
    qDebug() << "ctrlEnter pressed";
    toolProcess = new QProcess();
    QStringList cmdList;
    cmdList << "search"
            << "--onlyvisible"
            << "--class"          //通过类名查找进程
            << "VisualBoyAdvance" //指定进程的类名
            << "keydown"          //按键按下信号
            << "Return";
    toolProcess->execute("xdotool", cmdList); //调用xdotool 
}
```

---

## 页面切换细节问题

在最终全模块整合中，页面切换时，需要考虑如下细节问题

1. 需要考虑不同页面之间计时器的控制问题。  
    例如从主界面切换到游戏详情界面时，需要将主界面的定时器进行关闭，同时开启游戏详情界面的计时器以实现其一定时间无操作返回屏保界面的功能

    2. 需要考虑不同页面之间页面切换的问题。由于存在多次切换页面的需求。因此单纯的new方法将会带来大量的无用内存占用问题，并且会带来页面管理的困难。

解决方案：

1. 设计统一的计时器控制接口，在进行页面切换时，通过该接口统一控制前后两个页面的定时器开关

    ```c++
    void Main_Page::on_gameManagePb_clicked()
    {
        TimerContorl(false);//关闭自身的定时器
        gameManagePtr->show();//显示子页面
        hide();//隐藏当前界面
        gameManagePtr->TimerControl(true);//开启子页面的定时器
    }
    ```

2. 将所需要跳转的界面以指针方式传入对象的构造函数并作为成员变量存储。之后需要进行跳转的时候，直接访问成员变量进行控制。

    ```c++
    //创建实例对象时传入切换的页面指针并进行存储
    Main_Page::Main_Page(QWidget *parent, AD_Page *ad_pagePtr) : 
    QWidget(parent),
    ui(new Ui::Main_Page),
    adPage_ptr(ad_pagePtr),//记忆屏保界面的指针
    timer(new QTimer()),
    // 游戏管理子页面的初始化，将其作为变量存储，将自身指针作为参数传入
    gameManagePtr(new GameManage(0, this, adPage_ptr)),
    // 游戏详情子页面的初始化，将其作为变量存储，将自身指针作为参数传入
    detailPagePtr(new GameDetail(0, this, adPage_ptr))
    {
        setGeometry(0, 0, 1024, 600);
        ui->setupUi(this);
    }
    //通过访问成员变量进行页面切换 而无需创建新页面
    void Main_Page::slot_itemClicked(QListWidgetItem *item)
    {
        TimerContorl(false);
        hide();
        detailPagePtr->ShowGameDetail(item->text());
        qDebug() << "item double clicked" << item->text();
    }
    ```

‍
