<template>
    <div id="app">
        <Tree class="tree" :data="treeData" @nodeClick="handleNodeClick"/>
    </div>
</template>

<script>

import Tree from './components/G6-Tree.vue'

export default {
    name: 'App',
    components: {
        Tree
    },
    data() {
        return {
            // 数据若变动 需要设置唯一key
            treeData: {
                "name": "XX党总支",
                "children": [
                    {
                        "name": "第三党支部",
                        "children": [
                            {
                                "name": "党小组一"
                            },
                            {
                                "name": "党小组二"
                            },
                            {
                                "name": "党小组三"
                            }
                        ]
                    },
                    {
                        "name": "临时党支部一",
                        "children": [
                            {
                                "name": "党小组一"
                            },
                            {
                                "name": "党小组二"
                            },
                            {
                                "name": "党小组三"
                            }
                        ]
                    },
                    {
                        "name": "临时党支部二"
                    },
                    {
                        "name": "临时党支部三"
                    },
                    {
                        "name": "临时党支部四"
                    },
                    {
                        "name": "临时党支部五"
                    }
                ]
            }
        }
    },
    methods: {
        handleNodeClick(node) { // 点击事件
            console.log(node);
        }
    }
}
</script>

<style>
    html, body {
        padding: 0;
        margin: 0;
    }
    .tree {
        margin: 0 auto;
        text-align: center;
    }
</style>
