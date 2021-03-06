<template>
  <div :class="classes" :style="styles" ref="table" v-scroll="scroll">
    <table class="k-table-header" v-if="headerFixed && fixed" :style="`top:${fixedTop}px;width:${fixedWidth}px`">
      <thead>
        <tr>
          <th v-for="(item,index) in columns" :key="index" :style="`width:${colWidths[index]}px`">
            <div class="k-table-cell">
              <template v-if="item.type&&item.type=='selection'">
                <label for="k-checkbox-all">
                  <k-checkbox @change="checkAll" v-model="checkedAll">全选</k-checkbox>
                </label>
              </template>
              <template v-else>{{item.title}}</template>
            </div>
          </th>
        </tr>
      </thead>
    </table>
    <table ref="dom">
      <thead>
        <tr>
          <th v-for="(item,index) in columns" :key="index">
            <div class="k-table-cell">
              <template v-if="item.type&&item.type=='selection'">
                <label for="k-checkbox-all">
                  <k-checkbox @change="checkAll" v-model="checkedAll">全选</k-checkbox>
                </label>
              </template>
              <template v-else>{{item.title}}</template>
            </div>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(item,index) in data" :key="index">
          <td v-for="(sub,n) in columns" :key="n" :style="tdStyle(sub.textAlign)">
            <div v-if="sub.type&&sub.type=='selection'" class="k-table-cell">
              <label for="">
                <k-checkbox v-model="item.checked" @change="checkChange(item)"></k-checkbox>
              </label>
            </div>
            <template v-else-if="sub.type&&sub.type=='html'">
              <template v-if="isShowTip(item,sub)">
                <Tooltip :content="item[sub.key].replace(/<\/?.+?>/g,'')" breaked>
                  <div :style="`width:${sub.width||textMaxLength*12}px;`" class="k-table-cell k-table-cell-hidden" v-html="item[sub.key]"> </div>
                </Tooltip>
              </template>
              <div v-html="item[sub.key]" class="k-table-cell" v-else></div>
            </template>
            <div v-else-if="sub.render" class="k-table-cell" :style="`width:${sub.width}px;overflow:hidden;`">
              <Expand :render="sub.render" :row="item" :column="sub" :index="Number(index)"></Expand>
            </div>
            <template v-else>
              <template v-if="isShowTip(item,sub)">
                <Tooltip :content="item[sub.key]" breaked>
                  <div :style="`width:${sub.width||textMaxLength*12}px;`" class="k-table-cell k-table-cell-hidden"> {{item[sub.key]}}</div>
                </Tooltip>
              </template>
              <template v-else>
                <div :style="`width:${sub.width}px;`" :class="['k-table-cell',{'k-table-cell-hidden':sub.overflow=='hidden'}]"> {{item[sub.key]}}</div>
              </template>
            </template>
          </td>
        </tr>
      </tbody>
    </table>
    <div v-if="!data||data.length==0" class="no-data">{{noDataText}}</div>
  </div>
</template>
<script>
import { Checkbox } from "../checkbox";
import Tooltip from "../tooltip";
import Expand from "./expand.js";
import scroll from '../../directives/winScroll.js'
let copyData = (data) => {
  const t = Object.prototype.toString.call(data)
  let o = t === '[object Array]' ? [] : (t === '[object Object]' ? {} : data)

  if (t === '[object Array]') {
    for (let i = 0; i < data.length; i++) {
      o.push(copyData(data[i]));
    }
  } else if (t === '[object Object]') {
    for (let i in data) {
      o[i] = copyData(data[i]);
    }
  }
  return o;
}
export default {
  components: { Expand: Expand, "k-checkbox": Checkbox, Tooltip, },
  name: "Table",
  directives: { scroll },
  props: {
    bordered: Boolean,
    headerFixed: Boolean,
    mini: Boolean,
    noDataText: { type: String, default: "暂无数据..." },
    data: { type: Array, default: () => [] }, // 表格数据
    columns: { type: Array, default: () => [] }, // 表格类目
    textMaxLength: { type: Number, default: 0 },
    scrollbarFixed: Boolean
  },
  computed: {
    classes() {
      return [
        "k-table",
        {
          ["k-table-bordered"]: this.bordered,
          ["k-table-mini"]: this.mini,
          ["k-table-fixed"]: this.headerFixed
        }
      ];
    },
    styles() {
      let style = {}
      style.overflow = this.data.length == 0 ? 'hidden' : ''
      style.position = this.fixed ? 'relative' : ''
      style.height = this.scrollbarFixed ? this.barHeight : 'auto'
      return style
    }
  },
  data() {
    return {
      colWidths: [],
      fixed: false,
      fixedTop: 0,
      fixedWidth: 0,
      checkedAll: false,
      selectRow: [], //所有选择的数据
      barHeight: '',
      tableHeight: 0
    };
  },
  mounted() {
    this.tableHeight = this.$refs.table.getBoundingClientRect().height
  },
  watch: {
    data: {
      handler(r1, r2) {
        let count = this.data.filter(x => x.checked == true).length
        this.checkedAll = this.data.length && count == this.data.length
      },
      deep: true
    }
  },
  methods: {
    isShowTip(item, sub) {
      if (!item || !sub || !sub.key) return false
      return (this.textMaxLength && (item[sub.key] && item[sub.key].length > this.textMaxLength)) || sub.tooltip;
    },
    setWidths() {
      if (this.headerFixed && this.columns && this.columns.length && this.data && this.data.length) {
        let td = this.$refs.dom.childNodes[2].lastElementChild.childNodes
        let colWidths = []
        td.forEach(x => {
          colWidths.push(x.offsetWidth)
        })
        this.colWidths = colWidths
      }
    },
    scroll() {
      let t = this.$refs.table.getBoundingClientRect().top

      if (this.headerFixed) {
        this.fixedTop = t * -1;
        this.fixedWidth = this.$refs.dom.offsetWidth;
        if (this.headerFixed && t < 0) {
          this.setWidths()
        }
        this.fixed = t < 0
      }
      if (this.scrollbarFixed) {
        let b = this.$refs.table.getBoundingClientRect().bottom
        let clientHeight = document.body.clientHeight
        // b > 0: 表格不在可视区下方
        // clientHeight > t： 表格不在可视区上方
        if (clientHeight > t && b > 0) {
          this.barHeight = clientHeight - t + 'px'
          if (this.tableHeight < parseInt(this.barHeight)) {
            this.barHeight = this.tableHeight
          }
        }
      }
    },
    tdStyle(align) {
      let obj = {};
      if (align) obj["text-align"] = align;
      return obj;
    },
    checkChange(item) {
      this.$set(item, 'checked', item.checked)
      // this.data[index].checked = checked
      this.selectRow = this.data.filter(item => item.checked == true);
      this.$emit("selection", this.selectRow, item);
      this.checkedAll = this.data.length && this.selectRow.length == this.data.length
    },
    checkAll(checked) {
      this.data.forEach(item => {
        this.$set(item, 'checked', checked)
      })
      this.selectRow = checked ? JSON.parse(JSON.stringify(this.data)) : [];
      this.$emit("selection", this.selectRow);
    }
  }
};
</script>