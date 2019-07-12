<template>
  <div>
    <!-- 文本框 -->
    <Input :size="searchSize" v-if="column.searchType === 'input'" clearable placeholder="输入关键字查询" class="item-input" v-model="searchValueObj[column.key]" />
    <!-- 数字框 -->
    <InputNumber :size="searchSize" v-if="column.searchType === 'number'" class="item-number" v-model="searchValueObj[column.key]"></InputNumber>
    <!-- 数字范围 -->
    <div v-if="column.searchType === 'numberrange'" class="item-numberrange">
      <InputNumber :size="searchSize" v-model="searchValueObj[column.key + 'Begin']"></InputNumber>
      <span class="divide">~</span>
      <InputNumber :size="searchSize" v-model="searchValueObj[column.key + 'End']"></InputNumber>
    </div>
    <!-- lov -->
    <div class="item-lov" v-if="column.searchType === 'lov'" @click="showLov(item, index)">
      <Input :size="searchSize" readonly placeholder="请选择" icon="ios-search" v-model="searchValueObj[column.key]" />
    </div>
    <!-- 下拉框 -->
    <Select :size="searchSize" v-if="column.searchType === 'select'" v-model="searchValueObj[column.codeField]" class="item-select" @on-change="emitSearch">
      <Option v-for="(item1, index1) in column.options" :value="item1.value" :key="`select-${index}-option-${index1}`">{{ item1.desc }}</Option>
    </Select>
    <!-- 日期选择 -->
    <DatePicker :size="searchSize" v-if="column.searchType === 'date'" type="date" placeholder="选择日期" class="item-date" :value="searchValueObj[column.key]" @on-change="dateChange($event, column.key)"></DatePicker>
    <!-- 日期范围 -->
    <DatePicker :size="searchSize" v-if="column.searchType === 'daterange'" type="daterange" placeholder="选择日期范围" class="item-date" :value="searchValueObj[column.key]" @on-change="daterangeChange($event, column.key)"></DatePicker>
    <lovs ref="lovDom" :lovOptions="lovOptions" @on-confirm-select="confirmLovSelect"></lovs>
  </div>
</template>

<script>
import Lovs from '../lovs/lovs.vue'
import { _debounce } from '@/libs/tools'
export default {
  name: 'TablesSearch',
  components: {
    Lovs
  },
  props: {
    /**
     * @description 筛选面板配置
     */
    columns: {
      type: Array,
      default: []
    },
    /**
     * @description 筛选面板配置
     */
    cols: {
      type: Number
    },
    /**
     * @description 是否简单查询
     */
    isSimple: {
      type: Boolean,
      default: false
    },
    /**
     * @description 筛选面板尺寸
     */
    searchSize: {
      type: String,
      default: 'default'
    },
  },
  data () {
    return {
      searchValue: '', // 简单查询值
      searchValueObj: {}, // 查询对象
      currentSearchIndex: -1, // 当前查询项索引
      lovOptions: {}, // lov配置
      showSearch: true, // 显示查询面板
    }
  },
  created () {
    this.setSearchValueObj();
  },
  methods: {
    /**
     * @description 设置筛选面板查询对象
     */
    setSearchValueObj () {
      this.searchValueObj = {}
      this.columns.forEach(item => {
        if (item.searchable) {
          if (item.searchType === 'numberrange' || item.searchType === 'daterange') {
            this.$set(this.searchValueObj, item.key + 'Begin', null);
            this.$set(this.searchValueObj, item.key + 'End', null);
          }
          if (item.searchType === 'select' || item.searchType === 'lov') {
            this.$set(this.searchValueObj, item.codeField, '');
          }
          this.$set(this.searchValueObj, item.key, '');
        }
      })
      console.log('查询对象：' + JSON.stringify(this.searchValueObj))
    },
    /**
     * @description 查询日期选择框值改变
     */
    dateChange (event, key) {
      this.$set(this.searchValueObj, key, event);
      this.emitSearch();
    },
    /**
     * @description 查询日期选择框值改变
     */
    daterangeChange (event, key) {
      this.$set(this.searchValueObj, key, event);
      this.$set(this.searchValueObj, key + 'Begin', event[0]);
      this.$set(this.searchValueObj, key + 'End', event[1]);
      this.emitSearch();
    },
    /**
     * @description 显示lov
     */
    showLov (column, index) {
      this.currentSearchIndex = index;
      this.lovOptions = {
        multiple: false, // 是否多选
        modalTitle: '选择' + column.title, // 弹窗标题
        lovTableColumns: column.lovTableColumns, // 表格列配置
        api: column.lovApi, // 接口
        additionParam: column.lovAdditionParam || {}, // 额外参数
      }
      setTimeout(() => {
        this.$refs.lovDom.requestLovData();
      }, 100);
    },
    /**
     * @description 确认选择
     */
    confirmLovSelect (data) {
      const selectData = this.columns[this.currentSearchIndex].handlelovSelectData(data);
      this.$set(this.searchValueObj, this.columns[this.currentSearchIndex].codeField, selectData.code);
      this.$set(this.searchValueObj, this.columns[this.currentSearchIndex].key, selectData.value);
      this.emitSearch();
    },
    /**
     * @description 重置搜索条件查询
     */
    resetSearch () {
      this.setSearchValueObj()
      this.emitSearch();
    },
    /**
     * @description 搜索条件查询
     */
    emitSearch () {
      if (this.isSimple) {
        this.$emit('on-search', { searchValue: this.searchValue });
      } else {
        const searchValueObj = { ...this.searchValueObj };
        this.columns.forEach(item => { // 删掉范围初始字段
          if (item.searchable) {
            if (item.searchType === 'numberrange' || item.searchType === 'daterange') {
              delete searchValueObj[item.key];
            }
          }
        })
        this.$emit('on-search', searchValueObj);
      }
    },
    /**
     * @description 收缩状态切换
     */
    toggleShowSearch () {
      this.showSearch = !this.showSearch;
      this.$emit('on-collapse', {});
    },
    /**
     * @description 查询防抖
     */
    debouncedEmitSearchParams: _debounce(function () {
      this.emitSearch()
    }, 500),
  },
  computed: {
  },
  watch: {
    /**
     * @description 监听查询字段
     */
    searchValue (newVal, oldVal) {
      this.debouncedEmitSearchParams()
    }
  }
}
</script>

<style lang="less">
.simple-input {
  width: 40%;
}
.collapse {
  text-align: center;
  font-size: 24px;
  height: 5px;
  position: relative;
  .ivu-icon {
    position: absolute;
    top: -15px;
    cursor: pointer;
  }
}
.table-search {
  width: 100%;
  display: flex;
  .search-items {
    flex: 1;
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    .search-item {
      display: flex;
      margin-bottom: 10px;
      align-items: center;
      line-height: 1.1;
      .item-title {
        flex: 1;
        text-align: right;
        margin-right: 10px;
      }
      .item-input,
      .item-number,
      .item-numberrange,
      .item-lov,
      .item-select,
      .item-date {
        width: 70%;
      }
      .item-numberrange {
        display: flex;
        .ivu-input-number {
          width: 47%;
        }
        .divide {
          flex: 1;
          text-align: center;
        }
      }
    }
  }
  .search-buttons {
    button {
      margin-left: 10px;
    }
  }
}
</style>
