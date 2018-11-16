Canvas Group的组件， block raycast参数为true的时候，会接受raycast，该ui后面的元素响应点击的时候会知晓是透过其它物体的，
可以通过
UnityEngine.EventSystems.EventSystem.current.IsPointerOverGameObject()
来查询当前的点击事件是否透过了其它的Object

连接：
https://blog.csdn.net/rcfalcon/article/details/49743915
https://answers.unity.com/questions/822273/how-to-prevent-raycast-when-clicking-46-ui.html
