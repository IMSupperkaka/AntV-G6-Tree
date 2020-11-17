/*
 * @Author: shawn.huashiyun 
 * @Date: 2020-11-17 20:41:36 
 * @Last Modified by: shawn.huashiyun
 * @Last Modified time: 2020-11-17 21:22:56
 */
<template>
    <div ref="tree"></div>
</template>

<script>

import G6 from '@antv/g6';

const COLLAPSE_ICON = function COLLAPSE_ICON(x, y, r) {
    return [['M', x, y], ['a', r, r, 0, 1, 0, r * 2, 0], ['a', r, r, 0, 1, 0, -r * 2, 0], ['M', x + 2, y], ['L', x + 2 * r - 2, y]];
};
const EXPAND_ICON = function EXPAND_ICON(x, y, r) {
    return [['M', x, y], ['a', r, r, 0, 1, 0, r * 2, 0], ['a', r, r, 0, 1, 0, -r * 2, 0], ['M', x + 2, y], ['L', x + 2 * r - 2, y], ['M', x + r, y - r + 2], ['L', x + r, y + r - 2]];
};

// 自定义节点
G6.registerNode('tree-node', {
    // 绘制节点
    drawShape: function drawShape(cfg, group) {

        const [width, height] = cfg.size || [40, 40];

        const fontSize = 12;

        const rect = group.addShape('rect', {
            attrs: {
                fill: '#0d2574',
                stroke: '#3460bc',
                height: height,
                width: width,
                cursor: 'pointer'
            }
        });

        const content = cfg.name.replace(/(.{19})/g, '$1\n');

        const bbox = rect.getBBox();
        // 获取文字总长度
        const textLength = G6.Util.getTextSize(content, fontSize)[0];
        // 获取文字间的距离
        const letter = G6.Util.getLetterWidth(content, fontSize);

        Array.prototype.map.call(content, (text, index) => {
            group.addShape('text', {
                attrs: {
                    text: text,
                    x: bbox.width / 2 - letter / 2 - bbox.minX,
                    y: (bbox.height - textLength) / 2 + index * letter,
                    fontSize,
                    textAlign: 'center',
                    textBaseline: 'middle',
                    fill: '#fff',
                    cursor: 'pointer'
                }
            });
        })

        const hasChildren = cfg.children && cfg.children.length > 0;
        
        if (hasChildren) {
            group.addShape('marker', {
                attrs: {
                    x: bbox.minX + 5,
                    y: bbox.minX + bbox.height + 2 / 3 * 17 - 6,
                    r: 6,
                    symbol: COLLAPSE_ICON,
                    stroke: '#43b9fa',
                    lineWidth: 2,
                    cursor: 'pointer'
                },
                className: 'collapse-icon'
            });
        }

        rect.attr({
            x: bbox.minX - 4,
            y: bbox.minY - 6
        });
        
        return rect;
    },
    // 设置状态
    setState(name, value, item) {
        const group = item.getContainer();
        const shape = group.get('children')[0];
        if(name === 'selected') {
            if(value) {
                shape.attr('fill', 'red');
            } else {
                shape.attr('fill', '#0d2574');
            }
        }
    }
}, 'single-shape');

// 自定义边 vertical-horizontal-vertical
G6.registerEdge('vhv', {
    draw(cfg, group) {
        const startPoint = cfg.startPoint;
        const endPoint = cfg.endPoint;
        const shape = group.addShape('path', {
            attrs: {
                stroke: '#3460bc',
                path: [
                    ['M', startPoint.x, startPoint.y], // 移至起始点
                    ['L', startPoint.x, endPoint.y / 3 +  2 / 3 * startPoint.y], // 三分之一处
                    ['L', endPoint.x, endPoint.y / 3 +  2 / 3 * startPoint.y],   // 三分之二处
                    ['L', endPoint.x, endPoint.y] // 移至终点
                ]
            }
        });
        return shape;
    }
});

// 自定义behavior 用以解决点击节点会同时选中和展开的问题
// 参照 https://github.com/antvis/G6/issues/736
G6.registerBehavior('custom-collapse-expand', {
    getDefaultCfg() {
        return {
            onChange() {}
        };
    },
    getEvents() {
        return {
            'node:click': 'onNodeClick'
        };
    },
    onNodeClick(e) {
        // 如果点击的不是marker 不进行收缩和展开
        if (e.target.cfg.type != 'marker') {
            return;
        }
        const item = e.item;
        // 如果节点进行过更新，model 会进行 merge，直接改 model 就不能改布局，所以需要去改源数据
        const sourceData = this.graph.findDataById(item.get('id'));
        const children = sourceData.children;
        // 叶子节点的收缩和展开没有意义
        if (!children || children.length === 0) {
            return;
        }
        const collapsed = !sourceData.collapsed;
        if (!this.shouldBegin(e, collapsed)) {
            return;
        }
        sourceData.collapsed = collapsed;
        item.getModel().collapsed = collapsed;
        this.graph.emit('itemcollapsed', { item: e.item, collapsed });
        if (!this.shouldUpdate(e, collapsed)) {
            return;
        }
        try {
            this.onChange(item, collapsed);
        } catch (e) {
            console.warn('G6 自 3.0.4 版本支持直接从 item.getModel() 获取源数据(临时通知，将在之后版本清除)', e);
        }
        this.graph.refreshLayout();
    }
});

// 自定义选中事件 用以解决点击marker的时候也会触发选中的问题
// 简易实现 完整实现参照源码 https://github.com/antvis/G6/blob/master/src/behavior/click-select.ts
G6.registerBehavior('custom-click-select', {
    getEvents() {
        return {
            'node:click': 'onNodeClick'
        };
    },
    onNodeClick(e) {
        // 如果点击的是marker 不触发选中反选事件
        if (e.target.cfg.type == 'marker') {
            return;
        }
        const item = e.item;
        this.graph.setItemState(item, 'selected', !item.hasState('selected'));
    }
})

export default {
    name: 'tree',
    props: {
        data: {
            type: Object
        }
    },
    watch: {
        data() {
            if (this.graph) {
                this.render();
            }
        }
    },
    mounted() {
        const graph = new G6.TreeGraph({
            container: this.$refs.tree,
            width: window.innerWidth,
            height: window.innerHeight,
            modes: {
                default: [{
                    type: 'custom-collapse-expand',
                    onChange: function onChange(item, collapsed) {
                        var data = item.get('model');
                        var icon = item.get('group').findByClassName('collapse-icon');
                        if (collapsed) {
                            icon.attr('symbol', EXPAND_ICON);
                        } else {
                            icon.attr('symbol', COLLAPSE_ICON);
                        }
                        data.collapsed = collapsed;
                        return true;
                    }
                },'drag-canvas', 'zoom-canvas', 'custom-click-select']
            },
            layout: {
                type: "compactBox",
                direction: "TB",
                getHeight: () => { // 返回节点高度 以展示出边
                    return 100;
                },
                getWidth: () => {
                    return 30;
                },
                getHGap: () => { // 返回节点横向间距
                    return 10;
                }
            },
            defaultNode: {
                type: 'tree-node',
                size: [30, 100],
                anchorPoints: [[0.5, 1], [0.5, 0]]
            },
            defaultEdge: {
                type: 'vhv',
                style: {
                    stroke: '#A3B1BF'
                }
            },
        });
        // 抛出点击事件
        graph.on('node:click', ev=> {
            this.$emit('nodeClick', ev.item);
        });
        this.graph = graph;
        this.render();
    },
    methods: {
        render() {
            G6.Util.traverseTree(this.data, function(item, index) {
                item.id = item.key || index;
            });
            this.graph.data(this.data);
            this.graph.render();
            this.graph.fitView();
        }
    }
}
</script>
