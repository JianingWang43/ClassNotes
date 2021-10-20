# css note

## position

- position relative是相对于正常文件流中这个元素原本的位置进行定位。relative定位的元素，其原本位置仍然会占用页面流的一部分，之后的元素会在那之后定位。

- position absolute是相对于这个元素最近的一个祖先进行定位，该祖先满足：position的值必须是：relative、absolute、fixed、(sticky)，若没有这样的祖先则相对于body进行定位。偏移值由其top、bottom、left、right值确定（而绝对定位的元素若超出其父元素的边界，要想将溢出的部分隐藏，则，想隐藏在哪个祖先里，该祖先必须同时设置position:relative/absolute/fixed和overflow:hidden的值）。absolute定位的元素，将不会占用页面流。

## width height

- width height若使用百分比，是相对于这个元素最近的一个祖先，该祖先满足：position的值必须是：relative、absolute、fixed，(sticky)。若没有这样的祖先则相对于body

