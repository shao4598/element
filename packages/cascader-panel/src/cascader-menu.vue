<script>
import ElScrollbar from 'element-ui/packages/scrollbar';
import CascaderNode from './cascader-node.vue';
import Locale from 'element-ui/src/mixins/locale';
import { generateId } from 'element-ui/src/utils/util';
import emitter from 'element-ui/src/mixins/emitter';

export default {
  name: 'ElCascaderMenu',

  // 这里一定要添加这行
  componentName: 'ElCascaderMenu',

  mixins: [Locale, emitter],

  inject: ['panel'],

  components: {
    ElScrollbar,
    CascaderNode
  },

  props: {
    nodes: {
      type: Array,
      required: true
    },
    index: Number
  },

  data() {
    return {
      activeNode: null,
      hoverTimer: null,
      id: generateId(),
      // 添加以下两个
      checkAll: false,
      isIndeterminate: false
    };
  },

  computed: {
    isEmpty() {
      return !this.nodes.length;
    },
    menuId() {
      return `cascader-menu-${this.id}-${this.index}`;
    },
    isCheckAll() {
      let config = this.panel.config;
      // 父子关联的话，只有第一个cascader-menu有全选按钮
      // 父子不关联的话，所有的cascader-menu都有全选按钮
      return config.checkAll && (config.checkStrictly || (!config.checkStrictly && this.index === 0));
    }
  },

  created() {
    if (this.panel.config.checkAll && this.index === 0) {
      this.updateInDeterminate();
      this.$on('updateInDeterminate', this.updateInDeterminate);
    }
  },

  watch: {
    nodes() {
      this.isCheckAll && this.updateInDeterminate();
    }
  },

  methods: {
    handleExpand(e) {
      this.activeNode = e.target;
    },
    handleMouseMove(e) {
      const { activeNode, hoverTimer } = this;
      const { hoverZone } = this.$refs;

      if (!activeNode || !hoverZone) return;

      if (activeNode.contains(e.target)) {
        clearTimeout(hoverTimer);

        const { left } = this.$el.getBoundingClientRect();
        const startX = e.clientX - left;
        const { offsetWidth, offsetHeight } = this.$el;
        const top = activeNode.offsetTop;
        const bottom = top + activeNode.offsetHeight;

        hoverZone.innerHTML = `
          <path style="pointer-events: auto;" fill="transparent" d="M${startX} ${top} L${offsetWidth} 0 V${top} Z" />
          <path style="pointer-events: auto;" fill="transparent" d="M${startX} ${bottom} L${offsetWidth} ${offsetHeight} V${bottom} Z" />
        `;
      } else if (!hoverTimer) {
        this.hoverTimer = setTimeout(this.clearHoverZone, this.panel.config.hoverThreshold);
      }
    },
    clearHoverZone() {
      const { hoverZone } = this.$refs;
      if (!hoverZone) return;
      hoverZone.innerHTML = '';
    },

    renderEmptyText(h) {
      return (
        <div class="el-cascader-menu__empty-text">{ this.t('el.cascader.noData') }</div>
      );
    },
    renderNodeList(h) {
      const { menuId, nodes, checkAll, isIndeterminate, handleCheckAllChange, isCheckAll } = this;
      const { isHoverMenu, config } = this.panel;
      const events = { on: {} };

      if (isHoverMenu) {
        events.on.expand = this.handleExpand;
      }

      const nodeItems = nodes.map((node, index) => {
        const { hasChildren } = node;
        return (
          <cascader-node
            key={ node.uid }
            node={ node }
            node-id={ `${menuId}-${index}` }
            aria-haspopup={ hasChildren }
            aria-owns = { hasChildren ? menuId : null }
            { ...events }></cascader-node>
        );
      });

      return [
        isCheckAll &&
        <el-checkbox
          class="el-checkbox__checkAll"
          indeterminate={isIndeterminate}
          value={checkAll}
          onChange={handleCheckAllChange}
        >
        全选
        </el-checkbox>,
        [...nodeItems],
        isHoverMenu ? <svg ref='hoverZone' class='el-cascader-menu__hover-zone'></svg> : null
      ];
    },
    handleCheckAllChange() {
      const { nodes, panel } = this;
      // 反转全选按钮的状态
      this.checkAll = !this.checkAll;
      for (let i = 0; i < nodes.length; i++) {
        const node = nodes[i];
        // 触发当前cascader-menu的所有cascader-node组件的的doCheck方法，效果和背后逻辑与点击cascader-node是一样的。
        if (!node.isDisabled) node.doCheck(this.checkAll, true);
      }
      // 触发cascader-panel的方法，这方法主要是更新checkValue的值。
      panel.calculateMultiCheckedValue();
    },
    updateInDeterminate() {
      const { panel, nodes } = this;
      if (panel.config.checkAll) {
      // 记录checked：true的数量
        let counter = 0;
        // 记录disabled：true的数量，就是不可以点击的cascader-node
        let disabledCounter = 0;
        // 记录非全选状态的cascader-node的数据量
        let indeterminateCounter = 0;
        for (let i = 0; i < nodes.length; i++) {
          const node = nodes[i];
          if (!node.isDisabled) {
            node.checked && counter++;
            node.indeterminate && indeterminateCounter++;
          } else {
            disabledCounter++;
          }
        }
        // 全选按钮的状态有下面两个计算出来
        this.checkAll = counter === (this.nodes.length - disabledCounter) && counter > 0;
        this.isIndeterminate = this.checkAll ? false : indeterminateCounter > 0 || counter > 0;
      }
    }

  },

  render(h) {
    const { isEmpty, menuId } = this;
    const events = { nativeOn: {} };

    // optimize hover to expand experience (#8010)
    if (this.panel.isHoverMenu) {
      events.nativeOn.mousemove = this.handleMouseMove;
      // events.nativeOn.mouseleave = this.clearHoverZone;
    }

    return (
      <el-scrollbar
        tag="ul"
        role="menu"
        id={ menuId }
        class="el-cascader-menu"
        wrap-class="el-cascader-menu__wrap"
        view-class={{
          'el-cascader-menu__list': true,
          'is-empty': isEmpty
        }}
        { ...events }>
        { isEmpty ? this.renderEmptyText(h) : this.renderNodeList(h) }
      </el-scrollbar>
    );
  }
};
</script>
