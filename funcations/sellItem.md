# 关于**Lethal Company**出售**scrap**的代码分析。

------


> Version: **48**
>
> File: **DepositItemDesk.cs | HUDManager.cs** 
>
> Programming Langauge: **C#**


## 方法

### 放置物品

当Client把物品放到Counter时，游戏主线程会调用这个函数。

> 判断条件。
>
> **垃圾台上的垃圾数量 < 12**
>
> **没有播放爪子伸出的动画**
>
> **网络管理单例不为空**
>
> **操作是由本地玩家执行的**

```c#
public void PlaceItemOnCounter(PlayerControllerB playerWhoTriggered ) 
    //playerWhoTriggered 玩家控制器
{
    if (deskObjectsContainer.GetComponentsInChildren<GrabbableObject>().Length < 12 && !inGrabbingObjectsAnimation && GameNetworkManager.Instance != null && playerWhoTriggered == GameNetworkManager.Instance.localPlayerController)
    {
        //操作方向矢量来完成物品放置到柜台的动作
        Vector3 vector = RoundManager.RandomPointInBounds(triggerCollider.bounds);
        vector.y = triggerCollider.bounds.min.y;
        if (Physics.Raycast(new Ray(vector + Vector3.up * 3f, Vector3.down), out var hitInfo, 8f, 1048640, QueryTriggerInteraction.Collide))
        {
            vector = hitInfo.point;
        }

        vector.y += playerWhoTriggered.currentlyHeldObjectServer.itemProperties.verticalOffset;
        vector = deskObjectsContainer.transform.InverseTransformPoint(vector);
       
 // !!! 向服务器发送一个添加放置的垃圾的报文       AddObjectToDeskServerRpc(playerWhoTriggered.currentlyHeldObjectServer.gameObject.GetComponent<NetworkObject>());
        //完成放置的动作
        playerWhoTriggered.DiscardHeldObject(placeObject: true, deskObjectsContainer, vector, matchRotationOfParent: false);
        Debug.Log("discard held object called from deposit items desk");
    }
}
```

#### Server-Method

```c#
[ServerRpc(RequireOwnership = false)]
public void AddObjectToDeskServerRpc(NetworkObjectReference grabbableObjectNetObject)
{
    NetworkManager networkManager = base.NetworkManager;
    if ((object)networkManager == null || !networkManager.IsListening)
    {
        return;
    }

    //获取rpc状态，是客户端执行。
    if (__rpc_exec_stage != __RpcExecStage.Server && (networkManager.IsClient || networkManager.IsHost))
    {
        ServerRpcParams serverRpcParams = default(ServerRpcParams);
        FastBufferWriter bufferWriter = __beginSendServerRpc(4150038830u, serverRpcParams, RpcDelivery.Reliable);
        bufferWriter.WriteValueSafe(in grabbableObjectNetObject, default(FastBufferWriter.ForNetworkSerializable));
        __endSendServerRpc(ref bufferWriter, 4150038830u, serverRpcParams, RpcDelivery.Reliable);
    }

    if (__rpc_exec_stage != __RpcExecStage.Server || (!networkManager.IsServer && !networkManager.IsHost))
    {
        return;
    }
    
//是服务器执行
    if (grabbableObjectNetObject.TryGet(out lastObjectAddedToDesk))
    {
        //如果垃圾台上不包含这个垃圾
        if (!itemsOnCounter.Contains(lastObjectAddedToDesk.GetComponentInChildren<GrabbableObject>()))
        {
            //将其添加到集合中
            itemsOnCounterNetworkObjects.Add(lastObjectAddedToDesk);
            itemsOnCounter.Add(lastObjectAddedToDesk.GetComponentInChildren<GrabbableObject>());
            //将垃圾在柜台上显现
            AddObjectToDeskClientRpc(grabbableObjectNetObject);
            //设置垃圾回收的倒计时时间
            grabObjectsTimer = Mathf.Clamp(grabObjectsTimer + 6f, 0f, 10f);
            //如果门没有打开 且 老板没被惹怒 或者 老板没发怒
            if (!doorOpen && (!currentMood.mustBeWokenUp || timesHearingNoise >= 5f))
            {
                //执行客户端打开门
                OpenShutDoorClientRpc();
            }
        }
    }
    else
    {
        Debug.LogError("ServerRpc: Could not find networkobject in the object that was placed on desk.");
    }
}
```

#### Client - Method

```c#
[ClientRpc]
public void AddObjectToDeskClientRpc(NetworkObjectReference grabbableObjectNetObject)
{
    NetworkManager networkManager = base.NetworkManager;
    if ((object)networkManager == null || !networkManager.IsListening)
    {
        return;
    }
	//是服务器执行
    if (__rpc_exec_stage != __RpcExecStage.Client && (networkManager.IsServer || networkManager.IsHost))
    {
        //发送添加到柜台的数据包
        ClientRpcParams clientRpcParams = default(ClientRpcParams);
        FastBufferWriter bufferWriter = __beginSendClientRpc(3889142070u, clientRpcParams, RpcDelivery.Reliable);
        bufferWriter.WriteValueSafe(in grabbableObjectNetObject, default(FastBufferWriter.ForNetworkSerializable));
        __endSendClientRpc(ref bufferWriter, 3889142070u, clientRpcParams, RpcDelivery.Reliable);
    }
//是客户端执行
    if (__rpc_exec_stage == __RpcExecStage.Client && (networkManager.IsClient || networkManager.IsHost))
    {
        if (grabbableObjectNetObject.TryGet(out lastObjectAddedToDesk))
        {
            lastObjectAddedToDesk.gameObject.GetComponentInChildren<GrabbableObject>().EnablePhysics(enable: false);
        }
        else
        {
            Debug.LogError("ClientRpc: Could not find networkobject in the object that was placed on desk.");
        }
    }
}
```

### 完成出售

#### Server-Method

```c#
public void SellItemsOnServer()
{
    //如果不是服务器
    if (!base.IsServer)
    {
        return;
    }
	//将打开门的动画状态设置为开
    inSellingItemsAnimation = true;
    
    int num = 0; //用于计算垃圾价值的总和
    for (int i = 0; i < itemsOnCounter.Count; i++)
    {
        if (!itemsOnCounter[i].itemProperties.isScrap)
        {
            if (itemsOnCounter[i].itemUsedUp)
            {
            }
        }
        else
        {
            num += itemsOnCounter[i].scrapValue;
        }
    }
	
    // 垃圾价值的总和 * 今日公司的税率
    num = (int)((float)num * StartOfRound.Instance.companyBuyingRate);
 
    Terminal terminal = UnityEngine.Object.FindObjectOfType<Terminal>();
    // 飞船的总游戏币 + 垃圾价值的总和
    terminal.groupCredits += num;
    
    //向客户端发送售卖的报文
    SellItemsClientRpc(num, terminal.groupCredits, itemsOnCounterAmount, StartOfRound.Instance.companyBuyingRate);
    //三步走
    SellAndDisplayItemProfits(num, terminal.groupCredits);
}
```

#### Client-Method

```c#
[ClientRpc]
public void SellItemsClientRpc(int itemProfit, int newGroupCredits, int itemsSold, float buyingRate)
{
    NetworkManager networkManager = base.NetworkManager;
    if ((object)networkManager != null && networkManager.IsListening)
    {
        if (__rpc_exec_stage != __RpcExecStage.Client && (networkManager.IsServer || networkManager.IsHost))
        {
            //向客户端发送售卖的数据包
            ClientRpcParams clientRpcParams = default(ClientRpcParams);
            FastBufferWriter bufferWriter = __beginSendClientRpc(3628265478u, clientRpcParams, RpcDelivery.Reliable);
            BytePacker.WriteValueBitPacked(bufferWriter, itemProfit);
            BytePacker.WriteValueBitPacked(bufferWriter, newGroupCredits);
            BytePacker.WriteValueBitPacked(bufferWriter, itemsSold);
            bufferWriter.WriteValueSafe(in buyingRate, default(FastBufferWriter.ForPrimitives));
            __endSendClientRpc(ref bufferWriter, 3628265478u, clientRpcParams, RpcDelivery.Reliable);
        }

        if (__rpc_exec_stage == __RpcExecStage.Client && (networkManager.IsClient || networkManager.IsHost) && !base.IsServer)
        {
            itemsOnCounterAmount = itemsSold;
            StartOfRound.Instance.companyBuyingRate = buyingRate;
            SellAndDisplayItemProfits(itemProfit, newGroupCredits);
        }
    }
}
```

```c#
private void SellAndDisplayItemProfits(int profit, int newGroupCredits)
{
    //上文提到过，不再赘述
    UnityEngine.Object.FindObjectOfType<Terminal>().groupCredits = newGroupCredits;
    //gameStats 是 EndOfGameStats的对象 
    //scrapValueCollected是游戏结束 总收集到的垃圾
    StartOfRound.Instance.gameStats.scrapValueCollected += profit;
    //刷新完成指标
    TimeOfDay.Instance.quotaFulfilled += profit;
    //垃圾物品的集合
    GrabbableObject[] componentsInChildren = deskObjectsContainer.GetComponentsInChildren<GrabbableObject>();
    //如果上一个延迟接受的物品的程序不为空
    if (acceptItemsCoroutine != null)
    {	//则直接停止
        StopCoroutine(acceptItemsCoroutine);
    }
	
    //开启一个新的等待延迟接受的物品的携程程序
    acceptItemsCoroutine = StartCoroutine(delayedAcceptanceOfItems(profit, componentsInChildren, newGroupCredits));
    
    CheckAllPlayersSoldItemsServerRpc();
}
```

```c#
private IEnumerator delayedAcceptanceOfItems(int profit, GrabbableObject[] objectsOnDesk, int newGroupCredits)
{
    yield return new WaitUntil(() => !inGrabbingObjectsAnimation);
    noiseBehindWallVolume = 0.3f;
    yield return new WaitForSeconds(currentMood.judgementSpeed);
    if ((float)(profit / Mathf.Max(objectsOnDesk.Length, 1)) <= 3f && patienceLevel <= 2f)
    {
        System.Random random = new System.Random(objectsOnDesk.Length + newGroupCredits);
        if (!attacking && random.Next(0, 100) < 30)
        {
            Attack();
            yield return new WaitUntil(() => !attacking);
            yield return new WaitForSeconds(2f);
        }
    }
    else
    {
        patienceLevel += 3f;
    }

    //关闭门
    OpenShutDoor(open: false);
    yield return new WaitForSeconds(0.5f);
    //将声音设置为正常
    noiseBehindWallVolume = 1f;
    //发送HUD
    HUDManager.Instance.DisplayCreditsEarning(profit, objectsOnDesk, newGroupCredits);
    PlayRewardEffects(profit);
    yield return new WaitForSeconds(1.25f);
    MicrophoneSpeak();
    inSellingItemsAnimation = false;
    itemsOnCounterAmount = 0;
}
```

```c#
public void DisplayCreditsEarning(int creditsEarned, GrabbableObject[] objectsSold, int newGroupCredits)
{
    Debug.Log($"Earned {creditsEarned}; sold {objectsSold.Length} items; new credits amount: {newGroupCredits}");
    //垃圾
    List<Item> list = new List<Item>();
    for (int i = 0; i < objectsSold.Length; i++)
    {
        list.Add(objectsSold[i].itemProperties);
    }

    Item[] array = list.Distinct().ToArray();
    string text = "";
    int num = 0;
    int num2 = 0;
    for (int j = 0; j < array.Length; j++)
    {
        num = 0;
        num2 = 0;
        for (int k = 0; k < objectsSold.Length; k++)
        {
            if (objectsSold[k].itemProperties == array[j])
            {
                num += objectsSold[k].scrapValue;
                num2++;
            }
        }

        //价格拼接
        text += $"{array[j].itemName} (x{num2}) : {num} \n";
    }

    moneyRewardsListText.text = text;
    moneyRewardsTotalText.text = $"TOTAL: ${creditsEarned}";
    moneyRewardsAnimator.SetTrigger("showRewards");
    rewardsScrollbar.value = 1f;
    if (list.Count > 8)
    {
        if (scrollRewardTextCoroutine != null)
        {
            StopCoroutine(scrollRewardTextCoroutine);
        }

        scrollRewardTextCoroutine = StartCoroutine(scrollRewardsListText());
    }
}
```

```c#

[ServerRpc(RequireOwnership = false)]
public void CheckAllPlayersSoldItemsServerRpc()
{
    NetworkManager networkManager = base.NetworkManager;
    if ((object)networkManager == null || !networkManager.IsListening)
    {
        return;
    }
	
    //客户端的发包
    if (__rpc_exec_stage != __RpcExecStage.Server && (networkManager.IsClient || networkManager.IsHost))
    {
        ServerRpcParams serverRpcParams = default(ServerRpcParams);
        FastBufferWriter bufferWriter = __beginSendServerRpc(1114072420u, serverRpcParams, RpcDelivery.Reliable);
        __endSendServerRpc(ref bufferWriter, 1114072420u, serverRpcParams, RpcDelivery.Reliable);
    }
	
    if (__rpc_exec_stage != __RpcExecStage.Server || (!networkManager.IsServer && !networkManager.IsHost))
    {
        return;
    }

   //客户端收到已经出售的垃圾的报文的人数
    clientsRecievedSellItemsRPC++;
    if (clientsRecievedSellItemsRPC < GameNetworkManager.Instance.connectedPlayers)
    {
        return;
    }

    clientsRecievedSellItemsRPC = 0;
    for (int i = 0; i < itemsOnCounterNetworkObjects.Count; i++)
    {
        if (itemsOnCounterNetworkObjects[i].IsSpawned)
        {
            itemsOnCounterNetworkObjects[i].Despawn();
        }
    }
	//清理垃圾数据包
    itemsOnCounterNetworkObjects.Clear();
    //清理垃圾台数据包
    itemsOnCounter.Clear();
    //调用完成出售
    FinishSellingItemsClientRpc();
}
```

```c#
[ClientRpc]
public void FinishSellingItemsClientRpc()
{
    NetworkManager networkManager = base.NetworkManager;
    if ((object)networkManager != null && networkManager.IsListening)
    {
        if (__rpc_exec_stage != __RpcExecStage.Client && (networkManager.IsServer || networkManager.IsHost))
        {
            //向客户端发包
            ClientRpcParams clientRpcParams = default(ClientRpcParams);
            FastBufferWriter bufferWriter = __beginSendClientRpc(2469293577u, clientRpcParams, RpcDelivery.Reliable);
            __endSendClientRpc(ref bufferWriter, 2469293577u, clientRpcParams, RpcDelivery.Reliable);
        }

        if (__rpc_exec_stage == __RpcExecStage.Client && (networkManager.IsClient || networkManager.IsHost))
        {
            depositDeskAnimator.SetBool("GrabbingItems", value: false);
            inGrabbingObjectsAnimation = false;
        }
    }
}
```

