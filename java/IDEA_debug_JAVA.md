### Java 调试jar包:
启动时，增加启动参数, 在VMoption选项中添加:
"""
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=50064
"""

或者:

"""
-Xdebug
-Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=50064
"""


增加idea配置，启动配置选择 remote, debugger mode  选择 attach to remote jvm
