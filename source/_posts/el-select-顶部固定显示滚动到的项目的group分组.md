---
title: el-select 顶部固定显示滚动到的项目的group分组
date: 2024-12-25 21:53:02
tags:
---


## 前言

项目需要实现一个下拉框，除了需要对选项分组，还需要在头部展示当前滚动位置的分组信息，因为第一行数据处在“其他”，所以显示“其他”。效果如下

![](attachment/Pasted%20image%2020241120152911.png)

由于 element-ui 并没有相关的使用案例，所以就自己手搓一个实现。里面涉及到的 dom 操作是我以前很少接触的，所以记录一下。

## 功能拆分

由于数据量比较大，需要提前获取数据，组合成分组 option 的数据格式。

并且需要一个筛选方法，对展示的左右两侧数据都有响应。

官方提供了一个 header 插槽，但是难点在于如何绑定当前滚动到的 option 的分组信息到插槽里呢。这里就涉及到 vue 的 dom 操作。


## 获取 dom

首先我需要获取 selectRef，在 vue 3 组件式 APi 中，这是一个组件实例。他并不是一个 dom 对象。组件实例上有许多当前组件的属性和方法，也有一些带着 $符号的 vue 提供的属性，其中 $el 就是获取当前组件实例 dom 对象的属性。

![](attachment/Pasted%20image%2020241120154336.png)

经过查阅资料，其中的 popperRef 属性提供了下拉框的 dom，并且通过在页面观察滚动所在 div，我们可以获取到滚动层的 dom 节点，并添加事件监听。

```js
const watchScroll = () => {
  nextTick(() => {
    const popperRef = selectRef.value?.popperRef; // 获取下拉框 DOM
    if (popperRef) {
      const scrollContainer = popperRef.querySelector('.ep-scrollbar__wrap');
      scrollContainer.addEventListener('scroll', onScroll);
    }
  });
};
```


我们获取滚动 div 到下拉框顶部的距离，然后需要比较每一个 option 到顶部的距离，找到相同的那个 option，就可以知道位于下拉框第一个的 option 的 dom。获取他的内容就可以知道他的分组信息，并且绑定到一个响应式变量上，实现动态渲染。

```js
const onScroll = (event) => {
  const scrollTop = event.target.scrollTop;
  const groupElements = event.target.querySelectorAll('.ep-select-group__wrap');


  let visibleDomain = "";

  groupElements.forEach((group) => {
    const { offsetTop, clientHeight } = group;
    if (scrollTop >= offsetTop && scrollTop < offsetTop + clientHeight) {
      visibleDomain = group.querySelector('.ep-select-group__title')?.textContent?.trim();
    }
  });

  currentDomain.value = visibleDomain || "";
};
```

包括上述功能的完整代码如下

```vue
<template>
  <el-select
    ref="selectRef"
    v-model="projectName"
    placeholder="Select"
    size="default"
    style="width: 240px"
    filterable
    :filter-method="filter"
    clearable
    :loading="loading"
  >
    <template #header>
      <div style="font-size: var(--ep-font-size-base); color:var(--ep-text-color-regular); font-weight: 800;">
        数据域  : {{ currentDomain }}
      </div>
    </template>
    <el-option-group
      v-for="group in options"
      :key="group.label"
      :label="group.label"
    >
      <el-option
        v-for="item in group.options"
        :key="item.value"
        :label="item.label"
        :value="item.value"
      >
        <span style="float: left">{{ item.label }}</span>
        <span
          style="
          float: right;
          color: var(--el-text-color-secondary);
          font-size: 13px;
        "
        >
          {{ item.projectEname }}
        </span>
      </el-option>
    </el-option-group>
  </el-select>
</template>

<script setup>
//远程查询的select
import { EiInfo } from '@/utils/eiinfo.js';
import myApi from '@/api/index.js/';

const projectName = defineModel()



const originalOptions = ref([]); // 用于存储原始数据
//筛选后的工作空间列表
const options = ref([]);
const loading = ref(false);
const currentDomain = ref(""); // 当前滚动可见的分组
const selectRef = ref(null); // el-select 的引用

//挂载后加载工作空间
onMounted(() => {
      loadProjectInfo();
      watchScroll();
    },
);

//每次option变化，初始的数据域都是第一个option的group
watch(options,()=>{
  currentDomain.value = options.value[0].label
})


const watchScroll = () => {
  nextTick(() => {
    const popperRef = selectRef.value?.popperRef; // 获取下拉框 DOM
    if (popperRef) {
      const scrollContainer = popperRef.querySelector('.ep-scrollbar__wrap');
      scrollContainer.addEventListener('scroll', onScroll);
    }
  });
};

const onScroll = (event) => {
  const scrollTop = event.target.scrollTop;
  const groupElements = event.target.querySelectorAll('.ep-select-group__wrap');


  let visibleDomain = "";

  groupElements.forEach((group) => {
    const { offsetTop, clientHeight } = group;
    if (scrollTop >= offsetTop && scrollTop < offsetTop + clientHeight) {
      visibleDomain = group.querySelector('.ep-select-group__title')?.textContent?.trim();
    }
  });

  currentDomain.value = visibleDomain || "";
};


const filter = (query) => {
  if (loading.value) return;

  loading.value = true;

  // 如果有查询条件，基于原始数据进行筛选
  if (query) {
    const lowerQuery = query.toLowerCase();

    // 过滤分组和分组内的项目
    options.value = originalOptions.value
      .map((group) => {
        const filteredItems = group.options.filter((item) => {
          return (
            item.label.toLowerCase().includes(lowerQuery) ||
            item.projectEname.toLowerCase().includes(lowerQuery)
          );
        });

        // 如果当前分组有匹配项，保留分组
        return filteredItems.length > 0
          ? { ...group, options: filteredItems }
          : null; // 如果没有匹配项，返回 null
      })
      .filter(Boolean); // 去掉 null 的分组
  } else {
    // 如果没有查询条件，恢复为原始数据
    options.value = [...originalOptions.value];
  }

  loading.value = false;
};


const loadProjectInfo = () => {
  loading.value = true;
  const projectList = []
  const queryInfo = new EiInfo();
  queryInfo.set("isGroup",true)
  myApi.getProjectInfo(queryInfo).then(res => {
    const resBlock = res.getBlock('result');
    for (let i = 0; i < resBlock.rows.length; i++) {
      let domain = resBlock.getCell(i,"domain")
      let value = resBlock.getCell(i, 'projectName');
      let label = resBlock.getCell(i, 'projectAlias');
      let projectEname = resBlock.getCell(i, 'projectEname')
      projectList.push({
        value: value,
        label: label,
        projectEname:projectEname,
        domain:domain
      });
    }

    options.value = sortByVariable(projectList,"domain")
    originalOptions.value = options.value
    loading.value = false;
    if (options.value.length > 0) {
      projectName.value = options.value[0].options[0].value;
    }
  });
};

const sortByVariable = (_list, _val) => {
  const result = []; // 用于存放分组后的数据
  const groupMap = new Map(); // 使用 Map 提高性能

  _list.forEach((item) => {
    const key = item[_val];
    if (!groupMap.has(key)) {
      groupMap.set(key, []); // 初始化分组
      result.push({
        label: key,
        options: groupMap.get(key),
      });
    }
    groupMap.get(key).push(item); // 将当前项加入对应分组
  });

  return result;
};

</script>

<style scoped>
.stick-group{
  position: fixed;
}
</style>
```


## 优化空间

- 数据量大，获取数据到整合成特定结构耗时长
- 在滚动的时候频繁地渲染