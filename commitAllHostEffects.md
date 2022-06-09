
```typescript
function commitAllHostEffects() {
  while (nextEffect !== null) {
    recordEffect();
    const effectTag = nextEffect.effectTag;

    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }
    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    let primaryEffectTag = effectTag & (Placement | Update | Deletion);
    switch (primaryEffectTag) {
      case Placement: {
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;
        break;
      }
      case PlacementAndUpdate: {
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Deletion: {
        commitDeletion(nextEffect);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```
<a name="ySEoM"></a>
# commitPlacement（插入）
```typescript
function commitPlacement(finishedWork: Fiber): void {

  // Recursively insert all host nodes into the parent.
  // 找到真实DOM父节点对应的Fiber
  const parentFiber = getHostParentFiber(finishedWork);

  // Note: these two variables *must* always be updated together.
  let parent;
  let isContainer;

  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentFiber.stateNode;
      isContainer = false;
      break;
    case HostRoot:
      parent = parentFiber.stateNode.containerInfo;
      isContainer = true;
      break;
    case HostPortal:
      parent = parentFiber.stateNode.containerInfo;
      isContainer = true;
      break;
    default:
      invariant(
        false,
        'Invalid host parent fiber. This error is likely caused by a bug ' +
          'in React. Please file an issue.',
      );
  }
  if (parentFiber.effectTag & ContentReset) {
    resetTextContent(parent);
    parentFiber.effectTag &= ~ContentReset;
  }
  // 找到当前节点要插入在哪个真实的DOM节点之前的那个节点
  // 可能为null
  const before = getHostSibling(finishedWork);
  // We only have the top Fiber that was inserted but we need to recurse down its
  // children to find all the terminal nodes.
  let node: Fiber = finishedWork;
  while (true) {
    // 如果当前节点是一个真实的DOM节点或者文本节点
    if (node.tag === HostComponent || node.tag === HostText) {
      // 首先判断插到before节点之前的这个before节点存在不存在
      if (before) {
        // 如果这个before节点存在，并且当前节点还是一个容器节点
        if (isContainer) {
          insertInContainerBefore(parent, node.stateNode, before);
        } else {
          insertBefore(parent, node.stateNode, before);
        }
      } else {
        if (isContainer) {
          appendChildToContainer(parent, node.stateNode);
        } else {
          appendChild(parent, node.stateNode);
        }
      }
    } else if (node.tag === HostPortal) {
      
    } else if (node.child !== null) {
      // 当前节点不是真实DOM节点的Fiber对象,并且子节点存在
      // 继续循环，找到子节点的真实DOM节点，将它插入父节点
      node.child.return = node;
      node = node.child;
      continue;
    }
    // 说明当前节点插入完了 直接返回
    if (node === finishedWork) {
      return;
    }
    // 往上查找兄弟节点存在的那个节点
    while (node.sibling === null) {
      if (node.return === null || node.return === finishedWork) {
        return;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    // 继续循环，判断兄弟节点是不是真实DOM,节点
    // 如果是，将它插入到真实的父节点
    node = node.sibling;
  }
}
```
<a name="EWhgj"></a>
## getHostParentFiber
```typescript
function getHostParentFiber(fiber: Fiber): Fiber {
  let parent = fiber.return;
  while (parent !== null) {
    if (isHostParent(parent)) {
      return parent;
    }
    parent = parent.return;
  }
}
```
<a name="cLIGl"></a>
## getHostSibling
```typescript
// 这个方法要做的事就是：找到当前节点要插入到哪个兄弟节点之前
// 这里所指的兄弟节点是指在真实DOM树中的兄弟节点
function getHostSibling(fiber: Fiber): ?Instance {
  let node: Fiber = fiber;
  siblings: while (true) {
    // 这个while循环的意思就是找到第一个存在兄弟节点的Fiber就跳出while循环
    while (node.sibling === null) {
      if (node.return === null || isHostParent(node.return)) {
        // 到了host parent节点或者root根节点，还没有找到有兄弟节点的Fiber
        // 只能退出循环了
        return null;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    // 指针指向当前节点的兄弟节点
    node = node.sibling;
    // 如果当前找到的兄弟节点不是一个具有真实DOM的Fiber节点
    // 那么就要向下查找当前节点的Child节点，直到找到真实的DOM节点
    // 或者找不到跳出循环
    while (
      node.tag !== HostComponent &&
      node.tag !== HostText &&
      node.tag !== DehydratedSuspenseComponent
    ) {
      if (node.effectTag & Placement) {
        // 如果当前节点也是需要插入的，那么久继续当前大的循环，尝试找当前节点的兄弟节点
        continue siblings;
      }
      // 如果当前节点没有孩子节点，那就只能尝试兄弟节点找了
      // 跳出本轮循环，进入大的循环，继续查找兄弟节点
      if (node.child === null || node.tag === HostPortal) {
        continue siblings;
      } else {
        // 当前节点具有孩子节点，就查找当前节点的孩子节点，
        // 继续循环，查找真实的DOM节点
        node.child.return = node;
        node = node.child;
      }
    }
    // 跳出上面的循环后，说明找到了一个真实的节点,
    // 如果当前节点不需要插入，说明找到了我们想要的那个真实的DOM节点
    // 否则 还得继续大循环，查找当前节点的兄弟节点
    if (!(node.effectTag & Placement)) {
      // Found it!
      return node.stateNode;
    }
  }
}
```
<a name="QDO7B"></a>
## isHostParent
```typescript
function isHostParent(fiber: Fiber): boolean {
  return (
    fiber.tag === HostComponent ||
    fiber.tag === HostRoot ||
    fiber.tag === HostPortal
  );
}
```

<a name="vUVEx"></a>
# commitWork（更新）
<a name="UILoY"></a>
## HostComponent的更新
```typescript
    case HostComponent: {
      // 拿到Fiber对应的DOM
      const instance: Instance = finishedWork.stateNode;
      if (instance != null) {
        // Commit the work prepared earlier.
        const newProps = finishedWork.memoizedProps;
        // For hydration we reuse the update path but we treat the oldProps
        // as the newProps. The updatePayload will contain the real change in
        // this case.
        const oldProps = current !== null ? current.memoizedProps : newProps;
        const type = finishedWork.type;
        // TODO: Type the updateQueue to be specific to host components.
        // DOM节点接收的更新 是一个数组，i 为 key， i + 1 为value的形式
        const updatePayload: null | UpdatePayload = (finishedWork.updateQueue: any);
        finishedWork.updateQueue = null;
        if (updatePayload !== null) {
          // 提交更新
          commitUpdate(
            instance,
            updatePayload,
            type,
            oldProps,
            newProps,
            finishedWork,
          );
        }
      }
      return;
    }
```
<a name="Wq4bJ"></a>
### commitUpdate
```typescript
export function commitUpdate(
  domElement: Instance,
  updatePayload: Array<mixed>,
  type: string,
  oldProps: Props,
  newProps: Props,
  internalInstanceHandle: Object,
): void {
  // Update the props handle so that we know which props are the ones with
  // with current event handlers.
  // 每个DOM节点上都挂在了虚拟DOM所接收到props
  // 这里有更新了也要将当前DOM节点属性上挂载的那个更新一下
  updateFiberProps(domElement, newProps);
  // Apply the diff to the DOM node.
  // 更新属性到DOM节点上
  updateProperties(domElement, updatePayload, type, oldProps, newProps);
}
```
updateProperties最终调用的是updateDOMProperties
<a name="eUlsP"></a>
### updateDOMProperties
针对DOM节点的不同属性进行处理
```typescript
function updateDOMProperties(
  domElement: Element,
  updatePayload: Array<any>,
  wasCustomComponentTag: boolean,
  isCustomComponentTag: boolean,
): void {
  // TODO: Handle wasCustomComponentTag
  for (let i = 0; i < updatePayload.length; i += 2) {
    // 从 updatePayload 上获取key和value
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];
    if (propKey === STYLE) {
      setValueForStyles(domElement, propValue);
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      setInnerHTML(domElement, propValue);
    } else if (propKey === CHILDREN) {
      setTextContent(domElement, propValue);
    } else {
      setValueForProperty(domElement, propKey, propValue, isCustomComponentTag);
    }
  }
}
```
<a name="gLFMu"></a>
## HostText的更新
```typescript
    case HostText: {
      const textInstance: TextInstance = finishedWork.stateNode;
      const newText: string = finishedWork.memoizedProps;
      const oldText: string =
        current !== null ? current.memoizedProps : newText;
      commitTextUpdate(textInstance, oldText, newText);
      return;
    }
```
<a name="q3EzD"></a>
### commitTextUpdate
```typescript
export function commitTextUpdate(
  textInstance: TextInstance,
  oldText: string,
  newText: string,
): void {
  textInstance.nodeValue = newText;
}
```
<a name="RXX7U"></a>
# commitDeletion（删除）
```typescript
function commitDeletion(current: Fiber): void {
  if (supportsMutation) {
    // ReactDOM走这个
    unmountHostComponents(current);
  } else {
    // Detach refs and call componentWillUnmount() on the whole subtree.
    commitNestedUnmounts(current);
  }
  // 卸载Ref
  detachFiber(current);
}
```
<a name="x1NEK"></a>
## unmountHostComponents
```typescript
// 这个过程也是一个深度优先的过程
// 优先子节点，没有的话，就兄弟节点
// 当兄弟节点和子节点都执行完，在向上到父节点
function unmountHostComponents(current): void {
  // We only have the top Fiber that was deleted but we need to recurse down its
  // children to find all the terminal nodes.
  let node: Fiber = current;

  // Each iteration, currentParent is populated with node's host parent if not
  // currentParentIsValid.
  // 标记当前的parent是否是一个真实的DOM节点
  let currentParentIsValid = false;

  // Note: these two variables *must* always be updated together.
  let currentParent;
  let currentParentIsContainer;

  while (true) {
    if (!currentParentIsValid) {
      let parent = node.return;
      // 这个循环的做的事情，其实就是向Fiber树向上查找
      // 找到最近的为真实DOM父节点节点的Fiber
      findParent: while (true) {
        invariant(
          parent !== null,
          'Expected to find a host parent. This error is likely caused by ' +
            'a bug in React. Please file an issue.',
        );
        switch (parent.tag) {
          case HostComponent:
            currentParent = parent.stateNode;
            currentParentIsContainer = false;
            break findParent;
          case HostRoot:
            currentParent = parent.stateNode.containerInfo;
            currentParentIsContainer = true;
            break findParent;
          case HostPortal:
            currentParent = parent.stateNode.containerInfo;
            currentParentIsContainer = true;
            break findParent;
        }
        parent = parent.return;
      }
      currentParentIsValid = true;
    }

    if (node.tag === HostComponent || node.tag === HostText) {
      // 提交嵌套卸载
      commitNestedUnmounts(node);
      // After all the children have unmounted, it is now safe to remove the
      // node from the tree.
      // 执行真实DOM节点的removeChild操作
      if (currentParentIsContainer) {
        removeChildFromContainer(
          ((currentParent: any): Container),
          (node.stateNode: Instance | TextInstance),
        );
      } else {
        removeChild(
          ((currentParent: any): Instance),
          (node.stateNode: Instance | TextInstance),
        );
      }
      // Don't visit children because we already visited them.
    } else if (
      enableSuspenseServerRenderer &&
      node.tag === DehydratedSuspenseComponent
    ) {
      // Delete the dehydrated suspense boundary and all of its content.
      if (currentParentIsContainer) {
        clearSuspenseBoundaryFromContainer(
          ((currentParent: any): Container),
          (node.stateNode: SuspenseInstance),
        );
      } else {
        clearSuspenseBoundary(
          ((currentParent: any): Instance),
          (node.stateNode: SuspenseInstance),
        );
      }
    } else if (node.tag === HostPortal) {
      if (node.child !== null) {
        // When we go into a portal, it becomes the parent to remove from.
        // We will reassign it back when we pop the portal on the way up.
        currentParent = node.stateNode.containerInfo;
        currentParentIsContainer = true;
        // Visit children because portals might contain host components.
        node.child.return = node;
        node = node.child;
        continue;
      }
    } else {
      commitUnmount(node);
      // Visit children because we may find more host components below.
      if (node.child !== null) {
        node.child.return = node;
        node = node.child;
        continue;
      }
    }
    if (node === current) {
      return;
    }
    while (node.sibling === null) {
      if (node.return === null || node.return === current) {
        return;
      }
      node = node.return;
      if (node.tag === HostPortal) {
        // When we go out of the portal, we need to restore the parent.
        // Since we don't keep a stack of them, we will search for it.
        currentParentIsValid = false;
      }
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```
<a name="we6w2"></a>
## commitNestedUnmounts
```typescript
// 从当前节点开始，先执行当前节点的commitUnmount
// 然后在执行子节点的，如果子节点不存在，就执行兄弟节点
// 当兄弟节点不存在，在向上到父节点
// 这是一个深度优先的过程
function commitNestedUnmounts(root: Fiber): void {
  // While we're inside a removed host node we don't want to call
  // removeChild on the inner nodes because they're removed by the top
  // call anyway. We also want to call componentWillUnmount on all
  // composites before this host node is removed from the tree. Therefore
  // we do an inner loop while we're still inside the host node.
  let node: Fiber = root;
  while (true) {
    commitUnmount(node);
    // Visit children because they may contain more composite or host nodes.
    // Skip portals because commitUnmount() currently visits them recursively.
    if (
      node.child !== null &&
      // If we use mutation we drill down into portals using commitUnmount above.
      // If we don't use mutation we drill down into portals here instead.
      (!supportsMutation || node.tag !== HostPortal)
    ) {
      node.child.return = node;
      node = node.child;
      continue;
    }
    if (node === root) {
      return;
    }
    while (node.sibling === null) {
      if (node.return === null || node.return === root) {
        return;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```
<a name="VupDA"></a>
## commitUnmount
```typescript
// 执行相应的卸载，function组件执行hook的卸载
// class组件执行 componentWillUnmount 并且卸载ref等
// 真实的DOM节点卸载绑定的ref等
function commitUnmount(current: Fiber): void {
  onCommitUnmount(current);

  switch (current.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      const updateQueue: FunctionComponentUpdateQueue | null = (current.updateQueue: any);
      if (updateQueue !== null) {
        const lastEffect = updateQueue.lastEffect;
        if (lastEffect !== null) {
          const firstEffect = lastEffect.next;
          let effect = firstEffect;
          do {
            const destroy = effect.destroy;
            if (destroy !== undefined) {
              safelyCallDestroy(current, destroy);
            }
            effect = effect.next;
          } while (effect !== firstEffect);
        }
      }
      break;
    }
    case ClassComponent: {
      safelyDetachRef(current);
      const instance = current.stateNode;
      if (typeof instance.componentWillUnmount === 'function') {
        safelyCallComponentWillUnmount(current, instance);
      }
      return;
    }
    case HostComponent: {
      safelyDetachRef(current);
      return;
    }
    case HostPortal: {
      // TODO: this is recursive.
      // We are also not using this parent because
      // the portal will get pushed immediately.
      if (supportsMutation) {
        unmountHostComponents(current);
      } else if (supportsPersistence) {
        emptyPortalContainer(current);
      }
      return;
    }
  }
}
```
