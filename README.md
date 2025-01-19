# 表格行列选中功能的实现与组件设计

## 1. 功能

点击侧边栏和顶栏后选中行或列，以便后续操作（如删除、居中等）。

## 2. 实现方法

### Table 组件

```javascript
const [selCells, setSelCells] = useState < NodeEntry < TableCellNode > [] > [];
useEffect(() => {
  if (!store.editor) return;
  const cachedSelCells = store.CACHED_SEL_CELLS?.get(store.editor);

  cachedSelCells?.forEach((cell) => {
    const [cellNode] = cell;
    const cellDom = ReactEditor.toDOMNode(store.editor, cellNode);
    if (cellDom) {
      cellDom.classList.remove("selected-cell-td");
    }
  });

  selCells?.forEach((cell) => {
    const [cellNode] = cell;
    const cellDom = ReactEditor.toDOMNode(store.editor, cellNode);
    if (cellDom) {
      cellDom.classList.add("selected-cell-td");
    }
  });

  store.CACHED_SEL_CELLS.set(store.editor, selCells);
}, [JSON.stringify(selCells)]);
```

1. **监听 `selCells` 并处理选中样式**
   - 将被选中的 `selCells` 转为 DOM 节点。
   - 为其添加选中样式类名（`classList.add`），同时清除上次的选中样式（`classList.remove`）。
2. **使用 `CACHED_CEL_CELLS` 缓存选中状态**
   - 在 `store` 中添加 `CACHED_CEL_CELLS`。
   - 状态更新时只需要处理新选中的单元格和上次选中的单元格，不影响其他 DOM 节点。

---

### AbstractSideDiv 组件：确定选中的范围

```javascript
export function AbstractSideDiv(props: AbstractSideDivProps) {
  const { index, type, divStyle, getTableNode, setSelCells } = props;
  const isColumn = type === 'column';
  const { store } = useEditorStore();
  const tableSideDivRef = useRef<HTMLDivElement | null>(null);

  return (
    <>
      <div
        ref={tableSideDivRef}
        key={index}
        data-ignore-slate
        contentEditable={false}
        suppressContentEditableWarning
        style={{
          ...divStyle,
        }}
        onMouseDown={(e) => {
          e.stopPropagation();
          e.preventDefault();
          const tableSlateNode = getTableNode();
          if (tableSlateNode && index !== -1) {
            const tablePath = ReactEditor.findPath(
              store.editor,
              tableSlateNode,
            );
            const tableEntry = Editor.node(store.editor, tablePath);
            const len = isColumn
              ? (tableSlateNode.children as Array<any>).length
              : (tableSlateNode.children as Array<any>)[0].children.length;
            const startPath = isColumn
              ? [...tablePath, 0, index]
              : [tablePath[0], 1, index, 0];
            const endPath = isColumn
              ? [...tablePath, len - 1, index]
              : [tablePath[0], 1, index, len - 1];
            addSelection(store, tableEntry, startPath, endPath, setSelCells);
          }
        }}
      ></div>
    </>
  );
}
```

1. **获取表格在文档中的位置**

   - 使用 `ReactEditor.findPath(store.editor, tableSlateNode)` 获取表格位置。

2. **列处理逻辑**

   - 计算表格行数：`len = tableSlateNode.children.length`。
   - 选中起点：`[...tablePath, 0, index]`（列 `index` 的第一个单元格）。
   - 选中终点：`[...tablePath, len - 1, index]`（列 `index` 的最后一个单元格）。

3. **行处理逻辑**

   - 计算当前行的单元数：`len = tableSlateNode[0].children.length`。
   - 选中起点：`[tablePath[0], 1, index, 0]`（行 `index` 的第一个单元格）。
   - 选中终点：`[tablePath[0], 1, index, len - 1]`（行 `index` 的最后一个单元格）。

4. **传递选中范围信息**
   - 将起点路径、终点路径、表格节点等信息传递给 addSelection，完成选中范围的状态更新。 \*[addSelection]: 根据起点路径和终点路径，计算表格中被选中的单元格范围，并更新状态

---

### RowSideDiv 和 ColSideDiv 组件：

以 colsidediv 为例

```javascript
export function ColSideDiv(props: {
  tableRef: any;
  getTableNode: any;
  setSelCells: any;
}) {
  const { tableRef, getTableNode, setSelCells } = props;

  const tableDom = (tableRef as any)?.current?.childNodes[0];
  const [colDomArr, setColDomArr] = useState(
    tableDom ? Array.from(tableDom?.firstChild?.children || []) : [],
  );

  useEffect(() => {
    if (tableDom) {
      setColDomArr(Array.from(tableDom.firstChild.children || []));
    }
  }, [tableDom]);

  return (
    <>
      <div
        data-ignore-slate
        className="col-div-bar-inner ignore-toggle-readonly"
        style={{
          position: 'relative',
          display: 'block',
          borderRight: '1px solid #DFDFDF',
          zIndex: 100,
        }}
        contentEditable={false}
      >
        {colDomArr?.map((td: any, index: number) => {
          const colRect = td?.getBoundingClientRect();
          const leftPosition = colRect?.left || 0;

          return (
            <AbstractSideDiv
              key={index}
              index={index}
              type={'column'}
              divStyle={{
                position: 'absolute',
                top: -16,
                left: leftPosition - 30,
                width: colRect?.width || td?.clientWidth,
                height: '14px',
                zIndex: 101,
                backgroundColor: 'pink',
              }}
              getTableNode={getTableNode}
              setSelCells={setSelCells}
            />
          );
        })}
      </div>
    </>
  );
}
```

**列处理**

- 使用 `tableDom?.firstChild?.children` 获取表格第一行的所有列。
- 为每一列添加 `AbstractSideDiv`，并设置其宽度：
  ```javascript
  width: td?.getBoundingClientRect()?.width || td?.clientWidth;
  ```

**行处理**

- 使用 `tableDom?.children` 获取表格的所有行。
- 为每一行添加 `AbstractSideDiv`，并设置其高度：
  ```javascript
  height: tr?.getBoundingClientRect?.()?.height || tr?.clientHeight;
  ```

---

## 3. 补充

### api 解释

1. `ReactEditor.toSlateNode(editor, domNode)`：将 DOM 节点转换为 Slate 编辑器节点。  
   传入 `editor` 和 `domNode`，返回对应的 Slate 节点。 [ReactEditor](https://docs.slatejs.org/libraries/slate-react/react-editor)

2. `ReactEditor.toDOMNode(editor, slateNode)`：将 Slate 编辑器节点转换为 DOM 元素。  
   传入 `editor` 和 `slateNode`，返回对应的 DOM 元素。 [ReactEditor](https://docs.slatejs.org/libraries/slate-react/react-editor)

3. `ReactEditor.findPath(editor, node)`：返回指定 Slate 节点的路径。  
   传入 `editor` 和 `node`，返回该节点的路径。 [ReactEditor](https://docs.slatejs.org/libraries/slate-react/react-editor)

4. `Editor.node(editor, path)`：获取指定路径的 Slate 节点。  
   传入 `editor` 和 `path`，返回该路径下的节点。 [Editor](https://docs.slatejs.org/api/nodes/editor)

5. `Array.from(arrayLike)`：将类数组对象转换为数组。  
   传入类数组对象，返回一个新的数组。 [Array.from](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/from)

6. `e.stopPropagation()`：阻止事件冒泡。  
   传入事件对象，阻止该事件冒泡。 [stopPropagation](https://developer.mozilla.org/en-US/docs/Web/API/Event/stopPropagation)

7. `e.preventDefault()`：阻止事件的默认行为。  
   传入事件对象，阻止该事件的默认行为。 [preventDefault](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault)

### DOM 与 Slate 节点的区别

| 特性         | DOM 节点                       | Slate 节点                               |
| ------------ | ------------------------------ | ---------------------------------------- |
| **表示形式** | HTML 元素                      | JavaScript 对象                          |
| **位置**     | 存在于浏览器的渲染树中         | 存在于内存中，由 Slate 编辑器维护        |
| **用途**     | 显示内容并处理用户交互         | 描述文档逻辑结构，支持编辑操作           |
| **关系**     | 表示视觉上的层级关系           | 表示文档语义上的层级关系                 |
| **内容**     | 包含 HTML 和文本               | 以 `text` 属性或嵌套子节点的形式存储内容 |
| **事件响应** | 直接与用户交互（点击、拖拽等） | 通过 Slate 提供的 API 操作               |

转换方法

```javascript
const slateNode = ReactEditor.toSlateNode(editor, domNode);
const domNode = ReactEditor.toDOMNode(editor, slateNode);
```

---

### getBoundingClientRect() 和 clientHeight 的差异

|                    | `getBoundingClientRect()`                                  | `clientHeight`                                                      |
| ------------------ | ---------------------------------------------------------- | ------------------------------------------------------------------- |
| **定义**           | 返回元素的大小及其相对于视口的位置（一个矩形对象）。       | 返回元素的内部高度（包括内边距，不包括滚动条、边框和外边距）。      |
| **返回值类型**     | `DOMRect` 对象，包含 `width`、`height`、`top`、`left` 等。 | 数字，表示像素值。                                                  |
| **包含的内容**     | **元素的渲染大小**（包括 CSS 的缩放效果）。                | **元素的内容高度**，包括 `padding`，但不包括 `border` 和 `margin`。 |
| **是否受缩放影响** | 是，返回的值会受到 CSS 缩放或变换（如 `scale()`）的影响。  | 否，仅返回未缩放的实际高度。                                        |
| **计算方式**       | 通过渲染树计算，基于元素在视口中的实际位置。               | 通过布局树计算，基于元素的盒模型。                                  |
| **动态变化支持**   | 会实时更新，反映元素位置和尺寸的变化。                     | 会实时更新，但仅反映高度相关的变化。                                |
| **常见用途**       | - 获取元素相对于视口的位置，用于定位。                     | - 获取滚动容器的高度或元素内容的可视高度。                          |
|                    | - 处理拖拽、动画等需要精确位置的场景。                     | - 判断元素是否溢出容器等布局场景。                                  |
| **性能消耗**       | 较高，触发浏览器的重绘和回流（Reflow）。                   | 较低，通常不会触发回流。                                            |
