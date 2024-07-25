| [English (en-US)](./README.md) | **简体中文 (zh-CN)** |
| --- | --- |

<br>

# 实现延迟挂钩

## 什么是“延迟挂钩”和它带来的好处？

通常挂钩DLL中函数的做法需要先将对应的DLL加载到进程空间并定位它的地址（例如，使用[`LoadLibraryW`](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibraryw) + [`GetProcAddress`](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress)）。

对于为特定程序设计的钩子，通常它们的目标函数将迟早被调用，DLL也是进程需要的，所以早些加载对应的DLL没什么问题。而对于被注入到不同进程中的钩子（尤其是全局钩子），它们不知道各个进程是否需要此DLL，所以通常它们仍将DLL加载到各个进程空间并挂钩函数，即使进程本身并不想要这个DLL。

试想一下一个有不少依赖项的全局钩子试图挂钩各种系统DLL的函数，则会将所有涉及的DLL都带入到所有进程进行加载和初始化，开销极大。

“延迟挂钩”是此问题的一个好方案。即如果目标DLL已加载则立即执行挂钩，否则等到目标DLL加载到进程的时候挂钩。

## 技术方案

显然，实现“延迟挂钩”的关键是在第一时间获得加载DLL的通知。“[DLL加载通知](https://learn.microsoft.com/en-us/windows/win32/devnotes/dll-load-notification)”机制自NT6被引入，这正是我们需要的。

参考[LdrRegisterDllNotification](https://learn.microsoft.com/en-us/windows/win32/devnotes/ldrregisterdllnotification)函数，DLL加载（与卸载）的通知将被发送给由此函数注册的回调，并且DLL映射的内存区域在那时可用，同时我们可以进行挂钩。

尽管Microsoft Learning提示相关API可能会被更改或删除，但它们的用法一直没变，仅自NT6.1将所持的锁由`LdrpLoaderLock`变为了专用的`LdrpDllNotificationLock`。总之，请保持回调尽可能简单。

> [!TIP]
> 如果你想了解Windows上“[DLL加载通知](https://learn.microsoft.com/en-us/windows/win32/devnotes/dll-load-notification)”的内部实现，参考我为[ReactOS](https://github.com/reactos/reactos)贡献的[ReactOS PR #6795](https://github.com/reactos/reactos/pull/6795)。不要参考[WINE的实现](https://gitlab.winehq.org/wine/wine/-/commit/4c13e1765f559b322d8c071b2e23add914981db7)，因为它截至此文编写时存在错误，例如，它的`LdrUnregisterDllNotification`没有检查节点是否处于链表中就进行了移除。

## 在SlimDetours中使用“延迟挂钩”

[SlimDetours](https://github.com/KNSoft/KNSoft.SlimDetours)提供了`SlimDetoursDelayAttach`函数注册延迟挂钩，具体可参考该函数声明上方的注释以及[示例：DelayHook](../../../Source/Demo/DelayHook.c)。

该示例中，先调用了`SlimDetoursDelayAttach`注册对`User32.dll!EqualRect`API的延迟挂钩，并通过检查它和`LdrGetDllHandle`的返回值确认此时`User32.dll`并未加载：
```C
/* Register SlimDetours delay hook */
Status = SlimDetoursDelayAttach((PVOID*)&g_pfnEqualRect,
                                Hooked_EqualRect,
                                g_usUser32.Buffer,
                                g_asEqualRect.Buffer,
                                DelayAttachCallback,
                                NULL);
if (!NT_SUCCESS(Status))
{
    TEST_FAIL("SlimDetoursDelayAttach failed with 0x%08lX\n", Status);
    return;
} else if (Status != STATUS_PENDING)
{
    TEST_FAIL("SlimDetoursDelayAttach succeeded with 0x%08lX, which is not using delay attach\n", Status);
    return;
}

/* Make sure user32.dll is not loaded yet */
Status = LdrGetDllHandle(NULL, NULL, &g_usUser32, &hUser32);
if (NT_SUCCESS(Status))
{
    TEST_SKIP("user32.dll is loaded, test cannot continue\n");
    return;
} else if (Status != STATUS_DLL_NOT_FOUND)
{
    TEST_SKIP("LdrGetDllHandle failed with 0x%08lX\n", Status);
    return;
}
```

然后调用`LdrLoadDll`加载`User32.dll`：
```C
/* Load user32.dll now */
Status = LdrLoadDll(NULL, NULL, &g_usUser32, &hUser32);
if (!NT_SUCCESS(Status))
{
    TEST_SKIP("LdrLoadDll failed with 0x%08lX\n", Status);
    return;
}
```

此时若`User32.dll`成功加载，则之前注册的延迟挂钩应已挂钩完成，进而验证延迟挂钩回调被正确调用以及`User32.dll!EqualRect`函数被成功挂钩：
```C
/* Delay attach callback should be called and EqualRect is hooked successfully */
TEST_OK(g_bDelayAttach);
Status = LdrGetProcedureAddress(hUser32, &g_asEqualRect, 0, (PVOID*)&pfnEqualRect);
if (NT_SUCCESS(Status))
{
    TEST_OK(pfnEqualRect(&rc1, &rc2) == TRUE);
    TEST_OK(g_lEqualRectCount == 1);
} else
{
    TEST_SKIP("LdrGetProcedureAddress failed with 0x%08lX\n", Status);
}
```

<br>
<hr>

本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议 (CC BY-NC-SA 4.0)](http://creativecommons.org/licenses/by-nc-sa/4.0/) 进行许可。  
<br>
**[Ratin](https://github.com/RatinCN) &lt;[<ratin@knsoft.org>](mailto:ratin@knsoft.org)&gt;**  
*中国国家认证系统架构设计师*  
*[ReactOS](https://github.com/reactos/reactos)贡献者*
