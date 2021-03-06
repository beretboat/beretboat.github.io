---
layout: post
title: Cell-based NSTableView 到 view-based NSTableview 转换的一点经验
description: ""
modified: 2015-07-21
tags: [Cocoa NSTableView objc]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

在去年WWDC上，苹果在[What's New in Cocoa](https://developer.apple.com/videos/wwdc/2014/) section中正式deprecate cell-based `NSTableView`。虽然之前已经预料到早晚有这么一天，但是我们一直都没有将我们已有的cell-based table view转换为view-based。原因很简单，因为工作量太大了 － 我工作所在团队负责的产品是一个千万行代码级别的Mac软件，该软件有上百个对话框，约70个`NSTableview`/`NSOutlineView`。这些table view大都是在10.5～10.6时期写的，不少table view还被高度customized。在cell-based table view的年代，为了在一个table cell里面塞进去一个复杂点的空间，需要实现一个复杂的包含若干child cell的`NSCell`。

# 转换步骤

与iOS不同，`NSTableView`除了支持 delegates 外还支持强大的 binding。对于这两种不同的使用方式，view-based 转换的操作也有很大的不同。

### Cocoa Binding

* 打开view-based开关, 用xib的话在IB有个切换开关:

{:.table}
| Left-Aligned  | Center Aligned  | Right Aligned |
| :------------ |:---------------:| -----:|
| col 3 is      | some wordy text | $1600 |
| col 2 is      | centered        |   $12 |
| zebra stripes | are neat        |    $1 |
{: rules="groups"}

* 重新Binding:

{:.table}
| optional | Description | cell-based | view-based |
|--------|-------|--------|--------|
| required | Content binding   | N/A   | Bind table view's Content to ArrayController with Controller Key 'arrangedObjects'   |
| required | Value binding   | In each column, Value of the column is bound to ArrayController with Controller Key 'arrangedObjects' and a Model Key path   | Bind the Value of the control to table view with Model Key Path 'objectValue.key'   |
| optional | Selection binding   | [NSArrayController selectedObjects] works without Selection binding.   | In table view, bind the table's 'Selection Indexes' to the ArrayController with a Controller Key of 'SelectionIndexes'.   |
| optional | Sorting binding   | For each column, in IB Binding inspector, check 'Creates Sort Descriptor'.   | 1) Bind the table's Sort Descriptors binding to the array controller's 'sortDescriptors' property. 2) In 'Attributes inspector' of the table column, fill in 'Sort key' and 'Selector'(normally 'localizedCaseInsensitiveCompare:' for NSString object type).  |


### Delegates

1. 和UITableView类似，必须实现的delegate方法:
	
	```
	- (NSView *)tableView:(NSTableView *)tableView
	viewForTableColumn:(NSTableColumn *)tableColumn
		row:(NSInteger)row
	```
2. 除非有特殊需求，否则移除cell-based table中常见的取值/设置方法，如:
	
	```
	- (id)tableView:(NSTableView *)pTableViewObj 	objectValueForTableColumn:(NSTableColumn *)pTableColumn row:	(int)pRowIndex
	```
其它和cell相关的方法也一并移除，如给通过`NSTableColumn`的`dataCell`属性设置cell。
3. 方法

	```
	- (BOOL)tableView:(NSTableView *)aTableView
	shouldEditTableColumn:(NSTableColumn *)aTableColumn
   	           row:(NSInteger)rowIndex
	```
只在cell-based中会被调用，必须通过其它delegate方法或binding设置editable属性。

# 潜在的“坑”

* Table view delegate 类中的 `awakeFromNib` 方法在每次 `NSTableCellView` 被创建的时候都会被调用一次。所以不要override`awakeFromNib`。如果不得已要override，也不能在里面做不能被反复做的事情。
* 如果要让 cell view 里面的 text field 是灰的(disabled)的, 先把它设置为可编辑的(editable)。一个只读的 text field 就算被 disable 也不会变灰色。
* 当 table view 中的 text field 处于被编辑状态时，table selection 是无效的。所以，在需要取得 table 的 selection set 之前，需要先让 text field 退出编辑模式。常用强制 text field 退出编辑的方法是 `[self.window makeFirstResponder:nil]`
* formatter 不再支持 IBOutlet。需要设置 formatter 属性的话需要可以在

	```
	- (NSView *)tableView:(NSTableView *)tableView
	viewForTableColumn:(NSTableColumn *)tableColumn
		row:(NSInteger)row
	```
中做。


# References

* [Table View Programming Guide for Mac](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/TableView/Introduction/Introduction.html#//apple_ref/doc/uid/10000026i-CH1-SW1)
* [View Based NSTableView Basic to Advanced, WWDC 2011](https://developer.apple.com/videos/wwdc/2011/#120)