<template>
  <div class="editorBox" :class="{ hide: hide }">
    <!-- 标签页模式 -->
    <div v-if="show && isTabsMode" class="tabs-mode">
      <el-tabs v-model="activeTab" type="card" @tab-click="handleTabClick">
        <el-tab-pane 
          v-for="(item, index) in editorItemList" 
          :key="item.title" 
          :label="item.title" 
          :name="item.title"
        >
          <EditorItem
            ref="editorItemRefs"
            :title="item.title"
            :language="item.language"
            :codeTheme="codeTheme"
            :codeFontSize="codeFontSize"
            :content="item.content"
            :preprocessorList="preprocessorListMap[item.title]"
            :showAddBtn="item.showAddBtn"
            :dir="dir"
            :showAllAddResourcesBtn="['vue2', 'vue3'].includes(item.language)"
            :showHeader="showHeader"
            :readOnly="readOnly"
            @code-change="
              code => {
                codeChange(item, code)
              }
            "
            @preprocessor-change="
              p => {
                preprocessorChange(item, p)
              }
            "
            @add-resource="
              languageType => {
                addResource(languageType || item.title)
              }
            "
            @add-importmap="addImportmap(item)"
            @space-change="
              noSpace => {
                item.showTitle = noSpace
              }
            "
            @create-code-img="
              editor => {
                showCreateCodeImg(editor, item)
              }
            "
          ></EditorItem>
        </el-tab-pane>
      </el-tabs>
    </div>
    
    <!-- 原有的拖拽模式 -->
    <Drag v-if="show && !isTabsMode" :number="editorItemList.length" :dir="dir">
      <DragItem
        v-for="(item, index) in editorItemList"
        :key="item.title"
        :index="index"
        :disabled="item.disableDrag"
        :showTouchBar="item.showTouchBar"
        :title="item.showTitle ? item.title : ''"
      >
        <EditorItem
          :title="item.title"
          :language="item.language"
          :codeTheme="codeTheme"
          :codeFontSize="codeFontSize"
          :content="item.content"
          :preprocessorList="preprocessorListMap[item.title]"
          :showAddBtn="item.showAddBtn"
          :dir="dir"
          :showAllAddResourcesBtn="['vue2', 'vue3'].includes(item.language)"
          :showHeader="showHeader"
          :readOnly="readOnly"
          @code-change="
            code => {
              codeChange(item, code)
            }
          "
          @preprocessor-change="
            p => {
              preprocessorChange(item, p)
            }
          "
          @add-resource="
            languageType => {
              addResource(languageType || item.title)
            }
          "
          @add-importmap="addImportmap(item)"
          @space-change="
            noSpace => {
              item.showTitle = noSpace
            }
          "
          @create-code-img="
            editor => {
              showCreateCodeImg(editor, item)
            }
          "
        ></EditorItem>
      </DragItem>
    </Drag>
    
    <!-- 资源管理弹窗 -->
    <EditAssets ref="EditAssetsComp"></EditAssets>
    <!-- 生成代码图片 -->
    <CodeToImg
      ref="CodeToImgComp"
      :getThemeData="getThemeData"
      :codeTheme="codeTheme"
      :codeFontSize="codeFontSize"
    ></CodeToImg>
    <!-- importmap编辑 -->
    <EditImportMap
      :codeTheme="codeTheme"
      :codeFontSize="codeFontSize"
    ></EditImportMap>
  </div>
</template>

<script setup>
import {
  ref,
  defineProps,
  onMounted,
  computed,
  getCurrentInstance,
  watch,
  onUnmounted,
  nextTick
} from 'vue'
import { useStore } from 'vuex'
import EditorItem from '@/components/EditorItem.vue'
import Drag from './Drag.vue'
import DragItem from './DragItem.vue'
import { defaultEditorMap, preprocessorListMap } from '@/config/constants'
import { ElMessage, ElTabs, ElTabPane } from 'element-plus'
import { codeThemeList } from '@/config/codeThemeList'
import { base } from '@/config'
import * as monaco from 'monaco-editor/esm/vs/editor/editor.api'
import CodeToImg from '@/components/CodeToImg.vue'
import EditImportMap from './EditImportMap.vue'
import EditAssets from './EditAssets.vue'

// props
const props = defineProps({
  // 是否隐藏编辑器
  hide: {
    type: Boolean,
    default: false
  },
  // 排布方向
  dir: {
    type: String,
    default: 'h' // v（垂直）、h（水平）
  },
  // 要显示的编辑器列表
  showList: {
    type: Array,
    default() {
      return ['HTML', 'CSS', 'JS']
    } // 目前共有四种编辑器：'HTML'、 'CSS'、 'JS'、 'VUE'
  },
  // 是否要显示编辑器的头部
  showHeader: {
    type: Boolean,
    default: true
  },
  // 不要触发代码运行
  notRunCode: {
    type: Boolean,
    default: false
  },
  // 编辑器只读
  readOnly: {
    type: Boolean,
    default: false
  }
})

// hooks定义部分
// 初始化数据
const useInit = () => {
  const store = useStore()
  return {
    store,
    proxy: getCurrentInstance().proxy,
    editData: computed(() => store.state.editData), // 数据
    codeTheme: computed(() => store.state.editData.config.codeTheme), // 代码主题
    codeFontSize: computed(() => store.state.editData.config.codeFontSize) // 代码字号
  }
}

// 初始化编辑器列表
const useInitEditorList = ({ props, editData }) => {
  let show = ref(false)
  // 编辑器列表
  let editorItemList = ref([])

  // 初始化编辑器列表数据
  const initEditorItemList = () => {
    editorItemList = ref(
      props.showList.map(item => {
        if (typeof item === 'string') {
          return {
            ...defaultEditorMap[item],
            showTitle: false
          }
        } else {
          return {
            ...defaultEditorMap[item.title],
            ...item,
            showTitle: false
          }
        }
      })
    )
  }
  initEditorItemList()

  // 数据变化后重新初始化
  watch(
    () => {
      return props.showList
    },
    initEditorItemList,
    {
      deep: true
    }
  )

  // 设置编辑器列表初始数据
  const setInitData = () => {
    const code = editData.value.code
    Object.keys(code).forEach(type => {
      let index = getIndexByType(type)
      if (index === -1) {
        return
      }
      editorItemList.value[index].content = code[type].content
      editorItemList.value[index].language = code[type].language
    })
  }

  return {
    show,
    editorItemList,
    setInitData
  }
}

// 标签页模式相关
const useTabsMode = ({ store }) => {
  // 当前激活的标签
  const activeTab = ref('HTML')
  
  // 编辑器实例引用
  const editorItemRefs = ref([])
  
  // 判断是否是标签页模式
  const isTabsMode = computed(() => {
    return store.state.editData.config.layout === 'tabs'
  })
  
  // 处理标签点击事件
  const handleTabClick = (tab) => {
    nextTick(() => {
      // 触发自定义事件通知编辑器重新布局
      window.dispatchEvent(new CustomEvent('tab-change', { detail: tab.props.name }))
      
      // 获取当前激活的编辑器实例并调用其重新布局方法
      if (editorItemRefs.value && editorItemRefs.value.length > 0) {
        // 找到当前激活的编辑器
        const activeEditor = editorItemRefs.value.find(editor => {
          return editor && editor.title === tab.props.name
        })
        
        if (activeEditor && activeEditor.relayoutEditor) {
          // 延迟一点执行，确保DOM已完全更新
          setTimeout(() => {
            activeEditor.relayoutEditor()
            activeEditor.resize && activeEditor.resize()
          }, 50)
        }
      }
    })
  }
  
  return {
    activeTab,
    isTabsMode,
    handleTabClick,
    editorItemRefs
  }
}

// 处理主题
const useTheme = ({ codeTheme, proxy }) => {
  let themeData = null
  // 加载主题
  const loadTheme = async () => {
    try {
      if (!codeTheme.value) {
        return
      }
      let item = codeThemeList.find(item => {
        return item.value === codeTheme.value
      })
      if (!item) {
        return
      }
      if (item.custom) {
        // 该主题已加载，直接使用缓存
        if (item.loaded) {
          themeData = item.cache
        } else {
          // 未加载，则先加载
          themeData = await (
            await fetch(`${base}themes/${codeTheme.value}.json`)
          ).json()
          item.loaded = true
          item.cache = themeData
        }
        monaco.editor.defineTheme(codeTheme.value, themeData)
      }
      monaco.editor.setTheme(codeTheme.value)
      proxy.$eventEmitter.emit('set-theme', themeData)
    } catch (error) {
      console.log(error)
      ElMessage.error('主题加载失败，请重试')
    }
  }

  // 获取主题数据
  const getThemeData = () => {
    return themeData
  }

  // 监听设置代码主题
  watch(codeTheme, () => {
    loadTheme()
  })

  return {
    loadTheme,
    getThemeData
  }
}

// 代码运行
const useRunCode = ({ store, proxy }) => {
  // 布局
  const layout = computed(() => {
    return store.state.editData.config.layout
  })

  // 发送运行代码的通知
  const runCode = () => {
    if (props.notRunCode) return
    proxy.$eventEmitter.emit('run')
    if (layout.value === 'newWindowPreview') {
      proxy.$eventEmitter.emit('preview_window_run')
    }
  }

  proxy.$eventEmitter.on('run-code', runCode)

  // 开启关闭全能console后重新运行代码
  const openAlmightyConsole = computed(() => {
    return store.state.editData.config.openAlmightyConsole
  })
  watch(
    () => {
      return openAlmightyConsole.value
    },
    () => {
      runCode()
    }
  )

  return {
    runCode
  }
}

// 代码编辑器状态改变
const useEditorChange = ({
  setInitData,
  store,
  editorItemList,
  autoRun,
  runCode,
  editData,
  proxy
}) => {
  // 重新设置代码数据
  const resetCode = () => {
    setInitData()
    runCode()
  }
  proxy.$eventEmitter.on('reset_code', resetCode)
  onUnmounted(() => {
    proxy.$eventEmitter.off('reset_code', resetCode)
  })

  // 代码修改事件
  const codeChange = (item, code) => {
    store.commit('setCodeContent', {
      type: item.title,
      code
    })
    autoRun()
  }

  // 获取指定语言的数据
  const getIndexByType = type => {
    return editorItemList.value.findIndex(item => {
      return item.title === type
    })
  }

  // 修改预处理器
  const preprocessorChange = (item, p) => {
    let index = getIndexByType(item.title)
    editorItemList.value[index].language = p
    editorItemList.value[index].content =
      editData.value.code[item.title].content
    store.commit('setCodePreprocessor', {
      type: item.title,
      preprocessor: p
    })
    runCode()
  }

  return {
    codeChange,
    getIndexByType,
    preprocessorChange
  }
}

// 自动运行
const useAutoRun = ({ store, runCode }) => {
  let autoRunTimer = null
  const isAutoRun = computed(() => {
    return store.state.editData.config.autoRun
  })
  const autoRun = () => {
    if (!isAutoRun.value) {
      return
    }
    clearTimeout(autoRunTimer)
    autoRunTimer = setTimeout(() => {
      runCode()
    }, 1000)
  }
  return {
    autoRun
  }
}

// 生成代码图片
const CodeToImgComp = ref(null)
const useCodeToImg = () => {
  // 显示设置弹窗
  const showCreateCodeImg = (_editor, item) => {
    CodeToImgComp.value.showCreateCodeImg(_editor, item)
  }

  return {
    showCreateCodeImg
  }
}

// 资源管理
const EditAssetsComp = ref(null)
const useAssets = () => {
  const addResource = (...args) => {
    EditAssetsComp.value.addResource(...args)
  }
  const addImportmap = (...args) => {
    EditAssetsComp.value.addImportmap(...args)
  }

  return {
    addResource,
    addImportmap
  }
}

// created部分
const { store, editData, codeTheme, proxy, codeFontSize } = useInit()
const { show, editorItemList, setInitData } = useInitEditorList({
  props,
  editData
})
const { activeTab, isTabsMode, handleTabClick, editorItemRefs } = useTabsMode({ store })
const { loadTheme, getThemeData } = useTheme({ codeTheme, proxy })
const { runCode } = useRunCode({ store, proxy })
const { autoRun } = useAutoRun({ store, runCode })
const { getIndexByType, preprocessorChange, codeChange } = useEditorChange({
  setInitData,
  store,
  editorItemList,
  autoRun,
  runCode,
  editData,
  proxy
})
const { showCreateCodeImg } = useCodeToImg()
const { addResource, addImportmap } = useAssets()
onMounted(async () => {
  await loadTheme()
  setInitData()
  show.value = true
  runCode()
})
</script>

<style lang="less" scoped>
.editorBox {
  width: 100%;
  height: 100%;
  display: flex;

  &.hide {
    display: none;
  }
  
  .tabs-mode {
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    
    :deep(.el-tabs) {
      height: 100%;
      display: flex;
      flex-direction: column;
      
      .el-tabs__header {
        margin-bottom: 5px; /* 添加与内容区域的间距 */
        background-color: var(--editor-header-background);
      }
      
      .el-tabs__content {
        flex: 1;
        overflow: hidden;
        height: calc(100% - 45px); /* 调整高度以适应新增的间距 */
      }
      
      .el-tab-pane {
        height: 100%;
      }
      
      .el-tabs__header {
        border-bottom:1px solid var(--editor-header-title-color);

        .el-tabs__nav {
          border:1px solid var(--editor-header-title-color);
        }

        .el-tabs__item {
          border-left:1px solid var(--editor-header-title-color);
        }

        .el-tabs__item:first-child {
          border-left: none;
        }
      }
      
      .el-tabs__item {
        color: var(--editor-header-color);
        transition: all 0.3s;
        
        &:hover {
          color: var(--editor-header-title-color);
          background-color: rgba(255, 255, 255, 0.1); /* 添加hover效果 */
        }
        
        &.is-active {
          color: var(--editor-header-title-color);
          background-color: var(--editor-header-active-background, rgba(255, 255, 255, 0.2)); /* 高亮当前选中的tab */
          font-weight: bold;
          border-bottom: 0px solid var(--editor-header-title-color, #409eff); /* 添加底部边框强调 */
        }
      }
    }
  }
}

/deep/ .el-dialog__body {
  padding: 20px;
}
</style>
