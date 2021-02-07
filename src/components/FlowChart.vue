<template>
  <div class="wrapper" ref="wrapper">
    <div
      class="flow-node"
      v-for="node in nodes"
      :style="position(node)"
      :key="node.id"
    >
      <slot name="node" :node="node"> </slot>
    </div>
  </div>
</template>
<script lang="ts">
import {
  defineComponent,
  onMounted,
  ref,
  PropType,
  watchEffect,
  reactive,
} from "vue";
import { FillData, Marker, Path, StrokeData, Svg, SVG } from "@svgdotjs/svg.js";
const cols: any[][] = [];
const rows: any[][] = [];

export interface FlowNode {
  id: string | number;
  col: number;
  row: number;
  [prop: string]: any;
}
export interface LineSymbol {
  title?: string;
}
export interface Line {
  source: number | string;
  target: number | string;
  symbols?: LineSymbol[];
  color?: string;
}
export interface NodeMargin {
  x?: number;
  y?: number;
}
// 'a' means all
export type LineType = "a" | Array<number>;
export interface Grid {
  row?: LineType;
  col?: LineType;
}

export interface RectArea {
  startCol: number;
  startRow: number;
  endCol: number;
  endRow: number;
  stroke?: StrokeData;
  fill?: FillData;
}
export type alignType = "start" | "middle" | "end";
export interface FlowTextType {
  rowStart?: number;
  rowEnd?: number;
  colStart?: number;
  colEnd?: number;
  rotate?: boolean;
  rowAlign?: alignType;
  colAlign?: alignType;
  content: string;
  font?: Record<string, any>;
}

export default defineComponent({
  name: "FlowChart",
  props: {
    nodes: {
      type: Array as PropType<FlowNode[]>,
      required: true,
    },
    lines: {
      type: Array as PropType<Line[]>,
      required: true,
    },
    hoverLineColor: {
      required: false,
      type: String,
      default: "#000",
    },
    defaultLineStroke: {
      required: false,
      type: Object as PropType<StrokeData>,
      default: () => {
        return { width: 1, color: "#000" };
      },
    },
    colWidth: {
      type: Number,
      required: false,
      default: 150,
    },
    rowHeight: {
      type: Number,
      required: false,
      default: 100,
    },
    nodeWidth: {
      type: Number,
      required: false,
      default: 120,
    },
    nodeHeight: {
      type: Number,
      required: false,
      default: 80,
    },
    margin: {
      type: Object as PropType<NodeMargin>,
      required: false,
      default: () => {
        return { x: 0, y: 0 };
      },
    },
    grid: {
      type: Object as PropType<Grid>,
      required: false,
    },
    defaultGridStroke: {
      required: false,
      type: Object as PropType<StrokeData>,
      default: () => {
        return { dasharray: "5,5", width: 1, color: "#999" };
      },
    },
    rects: {
      type: Array as PropType<Array<RectArea>>,
      required: false,
    },
    defaultRectStroke: {
      required: false,
      type: Object as PropType<StrokeData>,
      default: () => {
        return { width: 1, color: "#000", dasharray: "5,5" };
      },
    },
    texts: {
      type: Array as PropType<Array<FlowTextType>>,
      required: false,
    },
    defaultTextFont: {
      required: false,
      type: Object,
      default: () => {
        return { color: "#000" };
      },
    },
    dis: {
      type: Number,
      required: false,
      default: 10,
    },
  },

  setup(props) {
    const {
      nodes,
      lines,
      defaultLineStroke,
      colWidth,
      rowHeight,
      nodeWidth,
      nodeHeight,
      margin,
      grid,
      rects,
      defaultRectStroke,
      texts,
      defaultTextFont,
      dis,
      hoverLineColor,
      defaultGridStroke,
    } = reactive(props);

    const arrowSize = 5;
    const arrowZoom = (arrowSize * (defaultLineStroke.width as number)) / 2;

    // 收集每一行节点数，节点按列排序
    const tempNodes = [...nodes];
    tempNodes.sort((a, b) => a.col - b.col);
    tempNodes.forEach(({ id, row }) => {
      if (typeof rows[row] === "undefined") {
        rows[row] = [];
      }
      rows[row].push(id);
    });

    // 收集每一列的节点数，节点间按行排列
    tempNodes.sort((a, b) => a.col - b.col);
    tempNodes.forEach(({ id, col }) => {
      if (typeof cols[col] === "undefined") {
        cols[col] = [];
      }
      cols[col].push(id);
    });

    const wrapper = ref<HTMLElement | null>(null);
    const offset = { x: margin.x || 0, y: margin.y || 0 };

    /**
     * 连接点算法
     */
    function rightMiddle(node: FlowNode, type: number) {
      let x = node.col * colWidth + nodeWidth;
      // type 1 represents end
      if (type) {
        x += arrowZoom;
      }
      return [x + offset.x, node.row * rowHeight + nodeHeight / 2 + offset.y];
    }
    function leftMiddle(node: FlowNode, type: number) {
      let x = node.col * colWidth;
      if (type) {
        x -= arrowZoom;
      }
      return [x + offset.x, node.row * rowHeight + nodeHeight / 2 + offset.y];
    }
    function bottomMiddle(node: FlowNode, type: number) {
      let y = node.row * rowHeight + nodeHeight;
      if (type) {
        y = y + arrowZoom;
      }
      return [node.col * colWidth + nodeWidth / 2 + offset.x, y + offset.y];
    }
    function topMiddle(node: FlowNode, type: number) {
      let y = node.row * rowHeight;
      if (type) {
        y = y - arrowZoom;
      }
      return [node.col * colWidth + nodeWidth / 2 + offset.x, y + offset.y];
    }

    /**
     * 给出每个节点的绝对位置
     */
    function position(item: FlowNode) {
      return {
        left: offset.x + item.col * colWidth + "px",
        top: offset.y + item.row * rowHeight + "px",
        width: nodeWidth + "px",
        height: nodeHeight + "px",
      };
    }

    /**
     * 根据节点给出节点上连接点位置
     */
    function points(n1: FlowNode, n2: FlowNode) {
      if (n2.col > n1.col) {
        return [rightMiddle(n1, 0), leftMiddle(n2, 1)];
      } else if (n2.col < n1.col) {
        return [leftMiddle(n1, 0), rightMiddle(n2, 1)];
      } else if (n2.row > n1.row) {
        return [bottomMiddle(n1, 0), topMiddle(n2, 1)];
      } else {
        return [topMiddle(n1, 0), bottomMiddle(n2, 1)];
      }
    }

    /**
     * 找出转折点，将其转为圆弧
     */
    const drawLineWithArc = (points: number[][]): string => {
      const len = points.length;
      if (len === 2) {
        return `M${points[0].join(" ")} L${points[1].join(" ")}`;
      } else if (len > 2) {
        let res = `M${points[0].join(" ")}`;
        for (let i = 1; i < len - 1; i++) {
          const [[x1, y1], [x2, y2], [x3, y3]] = [
            points[i - 1],
            points[i],
            points[i + 1],
          ];
          // 相连两点之间必在同一直线上，abs(x-x0) + abs(y-y0) 即为两点之间的直线距离
          const offset1 = Math.abs(x2 - x1) + Math.abs(y2 - y1);
          const offset2 = Math.abs(x3 - x2) + Math.abs(y3 - y2);
          const radius =
            Math.min(offset1, offset2) * 0.5 > 15
              ? 15
              : Math.min(offset1, offset2) * 0.5;
          // 同一竖线上
          if (x2 === x1) {
            // 当前点相对前节点下侧
            if (y2 > y1) {
              res += `L${x2} ${y2 - radius}`;

              // 下节点相对当前点右侧，逆时针圆弧
              if (x3 > x2) {
                res += `A${radius} ${radius} 0 0 0 ${x2 + radius} ${y2}`;
              }
              // 左侧，顺时针圆弧
              else {
                res += `A${radius} ${radius} 0 0 1 ${x2 - radius} ${y2}`;
              }
            } else {
              // 上侧
              res += `L${x2} ${y2 + radius}`;
              // 下节点相对当前点右侧，顺时针圆弧
              if (x3 > x2) {
                res += `A${radius} ${radius} 0 0 1 ${x2 + radius} ${y2}`;
              }
              // 左侧，逆时针圆弧
              else {
                res += `A${radius} ${radius} 0 0 0 ${x2 - radius} ${y2}`;
              }
            }
          } else {
            // 同一横线上
            // 右侧
            if (x2 > x1) {
              res += `L${x2 - radius} ${y2}`;
              // 下侧，顺时针
              if (y3 > y2) {
                res += `A${radius} ${radius} 0 0 1 ${x2} ${y2 + radius}`;
              } else {
                // 下侧，逆时针
                res += `A${radius} ${radius} 0 0 0 ${x2} ${y2 - radius}`;
              }
            } else {
              // 左侧
              res += `L${x2 + radius} ${y2}`;
              // 下侧，逆时针
              if (y3 > y2) {
                res += `A${radius} ${radius} 0 0 0 ${x2} ${y2 + radius}`;
              } else {
                // 下侧，顺时针
                res += `A${radius} ${radius} 0 0 1 ${x2} ${y2 - radius}`;
              }
            }
          }
        }
        res += `L${points[len - 1].join(" ")}`;
        return res;
      }
      return "";
    };

    /**
     * 根据起始节点和到达节点给出中间的折线连接节点
     */
    function linePath(line: Line): string {
      const sourceNode = nodes.find((item) => item.id === line.source);
      const targetNode = nodes.find((item) => item.id === line.target);

      if (sourceNode && targetNode) {
        const [[startX, startY], [endX, endY]] = points(sourceNode, targetNode);

        // 连接点距离
        let distance = dis;
        let col = sourceNode.col;
        let row = sourceNode.row;
        let endRow = targetNode.row;
        // 位于同一列处理
        if (startX === endX) {
          const sourceIndex = cols[col].findIndex(
            (val) => val === sourceNode.id
          );
          // 起点和终点在同一列且相邻
          if (
            targetNode.id === cols[col][sourceIndex + 1] ||
            targetNode.id === cols[col][sourceIndex - 1]
          ) {
            // return [[startX, startY], [endX, endY]]
            return drawLineWithArc([
              [startX, startY],
              [endX, endY],
            ]);
          }
          // 起点和终点在同一列不相邻
          // 位于第一列，则不能让连接线通过边界线
          if (col === 0) {
            col = 1;
          }
          if (endY < startY) {
            distance = -dis;
          }
          // return [[startX, startY], [startX, startY + distance], [col * colWidth, startY + distance], [col * colWidth, endY - distance], [endX, endY - distance], [endX, endY]]
          return drawLineWithArc([
            [startX, startY],
            [startX, startY + distance],
            [col * colWidth, startY + distance],
            [col * colWidth, endY - distance],
            [endX, endY - distance],
            [endX, endY],
          ]);
        }
        // 位于同一行处理
        if (startY === endY) {
          const sourceIndex = rows[row].findIndex(
            (val) => val === sourceNode.id
          );
          // 位于同一行且相邻
          if (
            targetNode.id === rows[row][sourceIndex + 1] ||
            targetNode.id === rows[row][sourceIndex - 1]
          ) {
            // return [[startX, startY], [endX, endY]]
            return drawLineWithArc([
              [startX, startY],
              [endX, endY],
            ]);
          }
          // 位于同一行不相邻
          // 位于第一行处理
          if (row === 0) {
            row = 1;
          }
          if (endX < startX) {
            distance = -dis;
          }
          // return [[startX, startY], [startX + distance, startY], [startX + distance, row * rowHeight], [endX - distance, row * rowHeight], [endX - distance, endY], [endX, endY]]
          return drawLineWithArc([
            [startX, startY],
            [startX + distance, startY],
            [startX + distance, row * rowHeight],
            [endX - distance, row * rowHeight],
            [endX - distance, endY],
            [endX, endY],
          ]);
        }

        // 不在同一行也不在同一列
        if (endX < startX) {
          distance = -dis;
        }
        // 如果两者在相邻列
        if (Math.abs(targetNode.col - sourceNode.col) === 1) {
          return drawLineWithArc([
            [startX, startY],
            [startX + distance, startY],
            [startX + distance, endY],
            [endX, endY],
          ]);
        }
        // 如果不在相邻列
        if (endY < startY) {
          endRow += 1;
        }
        // return [[startX, startY], [startX + distance, startY], [startX + distance, endY], [endX, endY]]
        return drawLineWithArc([
          [startX, startY],
          [startX + distance, startY],
          [startX + distance, endRow * rowHeight],
          [endX - distance, endRow * rowHeight],
          [endX - distance, endY],
          [endX, endY],
        ]);
      }
      // return []
      return "";
    }

    const changeLineColor = (path: Path, arrow: Marker, color: string) => {
      path.stroke({ color: color });
      path.marker("end", arrow);
    };

    const drawLines = (draw: Svg, lines: Line[]) => {
      // 连接线的箭头
      const arrow = draw.marker(arrowSize, arrowSize, function(add) {
        add.path(`M 0 0 L ${arrowSize} ${arrowSize / 2} L 0 ${arrowSize} z`);
      });
      const hoverArrow = draw
        .marker(arrowSize, arrowSize, function(add) {
          add.path(`M 0 0 L ${arrowSize} ${arrowSize / 2} L 0 ${arrowSize} z`);
        })
        .fill(hoverLineColor);

      // 连接线
      lines.forEach((line) => {
        const stroke = {
          ...defaultLineStroke,
          color: line.color || defaultLineStroke.color,
        };
        const computedColor = stroke.color as string;
        const pathes = linePath(line);
        const path = draw
          .path(pathes)
          .fill("none")
          .stroke(stroke)
          .attr({
            "stroke-linecap": "butt",
          });
        path.marker("end", arrow.fill(computedColor));
        path.mouseover((event: MouseEvent) => {
          changeLineColor(path, hoverArrow, hoverLineColor);
          event.stopPropagation();
          path.addTo(draw);
        });
        path.mouseout((event: MouseEvent) => {
          event.stopPropagation();
          changeLineColor(path, arrow, computedColor);
        });
      });
    };

    const drawRects = (
      draw: Svg,
      rects: RectArea[],
      wrapperWidth: number,
      wrapperHeight: number
    ) => {
      rects.forEach((rect) => {
        let startX = rect.startCol * colWidth;
        let startY = rect.startRow * rowHeight;
        let height = (rect.endRow - rect.startRow) * rowHeight;
        let width = (rect.endCol - rect.startCol) * colWidth;
        // 处理边界情况，不允许和 Svg 四周重合
        if (startX === 0) {
          startX += 1;
        }
        if (startY === 0) {
          startY += 1;
        }
        if (startX + width === wrapperWidth) {
          width -= 1;
        }
        if (startY + height === wrapperHeight) {
          height -= 1;
        }

        draw
          .rect(width, height)
          .stroke(defaultRectStroke)
          .fill("none")
          .move(startX, startY)
          .radius(10)
          .attr({ "shape-rendering": "crispEdges" });
      });
    };

    const drawGrid = (
      draw: Svg,
      wrapperWidth: number,
      wrapperHeight: number
    ) => {
      if (grid && grid.col) {
        if (typeof grid.col === "string") {
          for (let i = 0; i < wrapperWidth / colWidth; i++) {
            draw
              .line(i * colWidth, 0, i * colWidth, wrapperHeight)
              .stroke(defaultGridStroke)
              .attr({ "shape-rendering": "crispEdges" });
          }
        } else {
          grid.col.forEach((col) => {
            draw
              .line(col * colWidth, 0, col * colWidth, wrapperHeight)
              .stroke(defaultGridStroke)
              .attr({ "shape-rendering": "crispEdges" });
          });
        }
      }
      if (grid && grid.row) {
        if (typeof grid.row === "string") {
          for (let i = 0; i < wrapperHeight / rowHeight; i++) {
            draw
              .line(0, i * rowHeight, wrapperWidth, i * rowHeight)
              .stroke(defaultGridStroke)
              .attr({ "shape-rendering": "crispEdges" });
          }
        } else {
          grid.row.forEach((row) => {
            draw
              .line(0, row * rowHeight, wrapperWidth, row * rowHeight)
              .stroke(defaultGridStroke)
              .attr({ "shape-rendering": "crispEdges" });
          });
        }
      }
    };

    const drawTexts = (draw: Svg, texts: FlowTextType[]) => {
      texts.forEach((text) => {
        const {
          rowStart = 0,
          rowEnd = 0,
          colStart = 0,
          colEnd = 0,
          rotate = false,
          rowAlign = "middle",
          colAlign = "middle",
          content,
          font = defaultTextFont,
        } = text;
        console.log("text", content);

        const textContent = draw
          .text(content)
          .font({ ...defaultTextFont, ...font });
        const bbox = textContent.bbox();
        const width = bbox.width;
        const height = bbox.height;
        let cx = 0;
        let cy = 0;
        const gridLineWidth = defaultGridStroke.width || 0;
        if (rowStart < rowEnd) {
          if (colAlign === "start") {
            cy = rowStart * rowHeight + height / 2 + gridLineWidth;
          } else if (colAlign === "middle") {
            cy = ((rowStart + rowEnd) * rowHeight) / 2;
          } else {
            cy = rowEnd * rowHeight - height / 2 - gridLineWidth;
          }
        } else if (rowStart == rowEnd) {
          cy = height / 2 + gridLineWidth;
        }
        if (colStart < colEnd) {
          if (rowAlign === "start") {
            cx = colStart * colWidth + width / 2 + gridLineWidth;
          } else if (rowAlign === "middle") {
            cx = ((colStart + colEnd) * colWidth) / 2;
          } else {
            cx = colEnd * colWidth - width / 2 - gridLineWidth;
          }
        } else if (colStart < colEnd) {
          cx = width / 2 + gridLineWidth;
        }
        if (rotate) {
          const xDiff = width / 2 - height / 2;
          const yDiff = height / 2 - width / 2;
          if (rowAlign === "start") {
            cx -= xDiff;
          } else if (rowAlign === "end") {
            cx += xDiff;
          }
          if (colAlign === "start") {
            cy -= yDiff;
          } else if (colAlign === "end") {
            cy += yDiff;
          }
        }
        textContent.cx(cx).cy(cy);
        if (rotate) {
          textContent.rotate(90);
        }
      });
    };

    onMounted(() => {
      const typedWrapper = wrapper.value as HTMLElement;
      const wrapperWidth = typedWrapper.offsetWidth;
      const wrapperHeight = typedWrapper.offsetHeight;

      const draw = SVG()
        .addTo(typedWrapper)
        .size("100%", "100%");

      watchEffect(() => {
        draw.clear();

        // 画矩形
        if (rects && rects.length > 0) {
          drawRects(draw, rects, wrapperWidth, wrapperHeight);
        }

        // 画网格线
        drawGrid(draw, wrapperWidth, wrapperHeight);

        // 画文字
        if (texts) {
          drawTexts(draw, texts);
        }

        // 画连线
        drawLines(draw, lines);
      });
    });

    return { position, wrapper };
  },
});
</script>

<style lang="scss" scoped>
.wrapper {
  width: 100%;
  height: 100%;
  position: relative;
  box-sizing: border-box;
}

.tit {
  background: #1890ff;
  position: absolute;
  bottom: 0;
  width: 100%;
  text-align: center;
  color: #fff;
  padding: 3px 0;
  font-size: 14px;
}

line {
  border: solid 1px #1890ff;
}
.text {
  font-size: 14px;
  padding: 0 10px;
}
.flow-node {
  position: absolute;
  // z-index: 100;
  left: 0;
  top: 0;
}
</style>
