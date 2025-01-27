<template>
  <vs-card class="input-pane" :class="{'start-position': !nodes.length}" actionable>
    <template v-if="showPane || activatedLayer === 1">
      <div v-if="activatedLayer === 1" slot="header">
        <h3 class="text-gray-400">
          Create Node
        </h3>
      </div>
      <div v-if="activatedLayer > 1" slot="header">
        <h3 class="text-gray-400">
          Add Node To Layer {{ activatedLayer }}
        </h3>
      </div>
      <div v-if="!nodes.length">
        <vs-button class="w-full" type="gradient" @click="importNodes">Import</vs-button>
        <input
          id="import"
          ref="nodeImport"
          class="hidden"
          type="file"
          name=""
          @input="importCSV"
        >
        <div class="my-[20px] text-gray-400">
          OR
        </div>
      </div>

      <vs-input v-model="label" class="w-full" placeholder="Enter Node Name" @keyup.enter="handleAddNode" />
      <div slot="footer">
        <vs-row vs-justify="flex-end">
          <vs-button class="w-[100px]" color="success" type="gradient" @click="handleAddNode">Add</vs-button>
        </vs-row>
      </div>
    </template>

    <template v-if="showHintPane">
      <div v-if="activatedLayer > 1" slot="header">
        <h3 class="text-gray-400">
          Add Node To Layer {{ activatedLayer + 1 }}
        </h3>
      </div>
      <span style="color: rgb(222, 109, 131)">You need to generate layer {{ activatedLayer + 1 }} first</span>
    </template>
  </vs-card>
</template>

<script>
import csvMixin from '@/mixin/csv'
import NetNode from '@/factory/node'
import NetLink from '@/factory/link'
import { find } from 'lodash'
import { mapState } from 'vuex'
export default {
  name: 'InputPane',
  mixins: [csvMixin],
  data: () => ({
    label: '',
    closeness: 0,
  }),
  computed: {
    ...mapState('network', ['nodes', 'selectedNode']),
    ...mapState('layer', ['activatedLayer', 'totalLayer']),
    showPane () {
      let show = false
      if (this.activatedLayer > 1 && this.selectedNode) show = true
      if (this.selectedNode) {
        if (this.activatedLayer === this.selectedNode.layer) show = false
        if (this.activatedLayer !== this.selectedNode.layer + 1) show = false
      }
      return show
    },
    showHintPane () {
      return !this.showPane && this.selectedNode && this.activatedLayer > 1 && this.selectedNode.layer === this.activatedLayer && this.totalLayer <= this.selectedNode.layer
    },
  },
  methods: {
    handleAddNode () {
      if (!this.label.trim()) {
        return this.$vs.notify({
          icon: 'warning',
          title: 'Notice',
          text: 'Please enter node name',
          color: 'warning',
          position: 'top-center',
        })
      }

      if (this.checkRepeatNode()) {
        return this.$vs.notify({
          icon: 'warning',
          title: 'Notice',
          text: 'Node already existing',
          color: 'warning',
          position: 'top-center',
        })
      }

      this.addNode()
      this.addLink()

      this.$vs.notify({
        icon: 'check',
        title: 'New Node',
        text: this.label.trim(),
        color: 'success',
        position: 'top-center',
      })

      this.label = ''
      this.closeness = 0
    },

    addNode () {
      const label = this.label.trim()
      const node = new NetNode({
        id: `${this.activatedLayer}-${label.toLowerCase().replaceAll(' ', '_')}`,
        label: label.toLowerCase().replaceAll(' ', '_'),
        closeness: Number(this.closeness),
        layer: this.activatedLayer,
      })
      this.$store.commit('network/ADD_NODES', node)
    },

    addLink () {
      const label = this.label.trim()
      if (!this.selectedNode || this.activatedLayer === 1) return
      const link = new NetLink({
        source: `${this.activatedLayer}-${label.toLowerCase().replaceAll(' ', '_')}`,
        target: this.selectedNode.id,
        label: '',
      })
      this.$store.commit('network/ADD_LINKS', link)
    },

    checkRepeatNode () {
      return Boolean(
        find(this.nodes, {
          id: `${this.activatedLayer}-${this.label.trim()}`,
          layer: this.activatedLayer,
        }),
      )
    },
  },
}
</script>

<style scoped lang="postcss">
.input-pane {
@apply absolute left-0 right-0 bottom-20 m-auto  bg-gray-600 px-[20px];
@apply w-full max-w-[500px] min-w-[150px] ;
}

.start-position {
  @apply bottom-[45%];
}

.input-container {
  @apply grid gap-[10px] grid-cols-2;
}

.add-node-btn {
  @apply flex-shrink-0 ml-[10px] py-[7.5px] px-[10px] cursor-pointer rounded-md font-medium bg-emerald-300 text-center;
  @apply hover:bg-opacity-50;
  transition: 0.3s;
}

::v-deep .vs-inputx{
  @apply !text-center;
}

::v-deep .vs-con-input{
  @apply items-center justify-center;
  input {
    @apply h-[40px] ;

  }
  span {
    top: unset !important;
  }
}

.input-disabled {
  @apply pointer-events-none cursor-not-allowed;
}
</style>
