近期笔记本出现了剪切、新建、粘贴、重命名等操作，文件夹视图不会自动刷新的问题。手动F5会刷新。

根据网上提供的办法我的情况得到解决。

<i class="icon" id="error"> </i> ==修改注册表== 注册表路径：HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Update的UpdateMode，值改为：0（DWORD），不存在则新建。参考[这里](https://jingyan.baidu.com/article/d7130635d45a5013fcf47544.html)。`我这里测试没有效果`。

<i class="icon" id="error"> </i> ==Realtek== 控制面板—Realtek高清晰音频管理器–当插入设备时，开启自动弹出对话框。参考[这里](https://blog.csdn.net/angelo99/article/details/78289889)。`我这里测试没有效果`。

<i class="icon" id="check"> </i> ==重置文件夹== 这个方法解决了我的问题<i class="icon" id="smile"> </i>。参考[这里](https://www.winhelponline.com/blog/desktop-or-folders-not-refreshing-automatically-in-windows/)的“Reset folder views”方法。具体步骤如下：

1. Open a folder window
2. From the File menu, select **Folder and Search Options**
3. Select the **View** tab of the Folder Options dialog.
4. Click the Reset Folders button.
5. Close File Explorer.