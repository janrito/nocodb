<template>
  <div>
    <v-container fluid class="wrapper mb-3">
      <v-row>
        <v-col>
          <v-radio-group row hide-details dense v-model="type" @change="$refs.input.validate()" class="pt-0 mt-0">
            <v-radio value="hm" label="Has Many"></v-radio>
            <v-radio value="mm" label="Many To Many"></v-radio>
            <v-radio disabled value="oo" label="One To One"></v-radio>
          </v-radio-group>
        </v-col>
      </v-row>
    </v-container>

    <v-container fluid class="wrapper">
      <v-row>
        <v-col cols="12">
          <v-autocomplete
            ref="input"
            outlined
            class="caption"
            hide-details="auto"
            :loading="isRefTablesLoading"
            label="Child Table"
            :full-width="false"
            v-model="relation.childTable"
            :items="refTables"
            item-text="_tn"
            item-value="tn"
            required
            dense
            :rules="tableRules"
          ></v-autocomplete>
        </v-col
        >
        <!--    <v-col cols="6">
              <v-text-field
                outlined
                class="caption"
                hide-details
                label="Child Column"
                :full-width="false"
                v-model="relation.childColumn"
                required
                dense
                ref="childColumnRef"
                @change="onColumnSelect"
              ></v-text-field>
            </v-col
            >-->
      </v-row>
      <template v-if="!isSQLite">
        <v-row>
          <v-col cols="6">
            <v-autocomplete
              outlined
              class="caption"
              hide-details
              label="On Update"
              :full-width="false"
              v-model="relation.onUpdate"
              :items="onUpdateDeleteOptions"
              required
              dense
              :disabled="relation.type !== 'real'"
            ></v-autocomplete>
          </v-col>
          <v-col cols="6">
            <v-autocomplete
              outlined
              class="caption"
              hide-details
              label="On Delete"
              :full-width="false"
              v-model="relation.onDelete"
              :items="onUpdateDeleteOptions"
              required
              dense
              :disabled="relation.type !== 'real'"
            ></v-autocomplete>
          </v-col>
        </v-row>


        <v-row>

          <v-col>
            <v-checkbox
              false-value="real"
              true-value="virtual"
              label="Virtual Relation"
              :full-width="false"
              v-model="relation.type"
              required
              class="mt-0"
              dense
            ></v-checkbox>
          </v-col>
        </v-row>
      </template>
    </v-container>
  </div>
</template>

<script>
export default {
  name: "linked-to-another-options",
  props: ['nodes', 'column', 'meta', 'isSQLite', 'alias'],
  data: () => ({
    type: 'hm',
    refTables: [],
    refColumns: [],
    relation: {},
    isRefTablesLoading: false,
    isRefColumnsLoading: false,
  }),
  async created() {
    await this.loadTablesList();
    this.relation = {
      childColumn: `${this.meta.tn}_id`,
      childTable: this.nodes.tn,
      parentTable: this.column.rtn || "",
      parentColumn: this.column.rcn || "",
      onDelete: "NO ACTION",
      onUpdate: "NO ACTION",
      updateRelation: this.column.rtn ? true : false,
      type: 'real'
    }
  },
  computed: {
    onUpdateDeleteOptions() {
      if (this.isMSSQL) {
        return ["NO ACTION"]
      }
      return [
        "NO ACTION",
        "CASCADE",
        "RESTRICT",
        "SET NULL",
        "SET DEFAULT"
      ];
    },
    tableRules() {
      return [
        v => !!v || 'Required',
        v => {
          if (this.type === 'mm')
            return !(this.meta.manyToMany || [])
                .some(mm => mm.tn === v && mm.rtn === this.meta.tn || mm.rtn === v && mm.tn === this.meta.tn)
              || 'Duplicate many to many relation is not allowed at the moment';
          if (this.type === 'hm')
            return !(this.meta.hasMany || [])
                .some(hm => hm.tn === v)
              || 'Duplicate has many relation is not allowed at the moment';
        },
      ]
    }
  },
  methods: {
    async loadColumnList() {
      this.isRefColumnsLoading = true;
      const result = await this.$store.dispatch('sqlMgr/ActSqlOp', [{
        env: this.nodes.env,
        dbAlias: this.nodes.dbAlias
      }, 'columnList', {tn: this.meta.tn}])


      const columns = result.data.list;
      this.refColumns = JSON.parse(JSON.stringify(columns));

      if (this.relation.updateRelation && !this.relationColumnChanged) {
        //only first time when editing add defaault value to this field
        this.relation.parentColumn = this.column.rcn;
        this.relationColumnChanged = true;
      } else {
        //find pk column and assign to parentColumn
        const pkKeyColumns = this.refColumns.filter(el => el.pk);
        this.relation.parentColumn = (pkKeyColumns[0] || {}).cn || "";
      }
      this.onColumnSelect();

      this.isRefColumnsLoading = false;
    },
    async loadTablesList() {
      this.isRefTablesLoading = true;


      const result = await this.$store.dispatch('sqlMgr/ActSqlOp', [{
        env: this.nodes.env,
        dbAlias: this.nodes.dbAlias
      }, 'tableList']);


      this.refTables = result.data.list.map(({tn, _tn}) => ({tn, _tn}))
      this.isRefTablesLoading = false;
    },
    async saveManyToMany() {
      try {
        await this.$store.dispatch('sqlMgr/ActSqlOpPlus', [
          {
            env: this.nodes.env,
            dbAlias: this.nodes.dbAlias
          },
          'xcM2MRelationCreate',
          {
            _cn: this.alias,
            ...this.relation,
            type: this.isSQLite || this.relation.type === 'virtual' ? 'virtual' : 'real',
            parentTable: this.meta.tn,
            updateRelation: this.column.rtn ? true : false
          }
        ]);
      } catch (e) {
        throw e
      }
    },
    async saveRelation() {
      if (this.type === 'mm') {
        await this.saveManyToMany();
        return;
      }
      try {
        const parentPK = this.meta.columns.find(c => c.pk);


        const childTableData = await this.$store.dispatch('sqlMgr/ActSqlOp', [{
          env: this.nodes.env,
          dbAlias: this.nodes.dbAlias
        }, 'tableXcModelGet', {
          tn: this.relation.childTable
        }]);

        const childMeta = JSON.parse(childTableData.meta)

        const newChildColumn = {};


        Object.assign(newChildColumn, {
          cn: this.relation.childColumn,
          _cn: this.relation.childColumn,
          rqd: false,
          pk: false,
          ai: false,
          cdf: null,
          dt: parentPK.dt,
          dtxp: parentPK.dtxp,
          dtxs: parentPK.dtxs,
          un: parentPK.un,
          altered: 1
        });

        const columns = [...childMeta.columns, newChildColumn];

        let result = await this.$store.dispatch('sqlMgr/ActSqlOpPlus', [{
          env: this.nodes.env,
          dbAlias: this.nodes.dbAlias
        }, "tableUpdate", {
          tn: childMeta.tn,
          _tn: childMeta._tn,
          originalColumns: childMeta.columns,
          columns
        }]);


        await this.$store.dispatch('sqlMgr/ActSqlOpPlus', [
          {
            env: this.nodes.env,
            dbAlias: this.nodes.dbAlias
          },
          this.relation.type === 'real' && !this.isSQLite ? "relationCreate" : 'xcVirtualRelationCreate',
          {
            ...this.relation,
            parentTable: this.meta.tn,
            parentColumn: parentPK.cn,
            updateRelation: this.column.rtn ? true : false,
            type: 'real'
          }
        ]);
      } catch (e) {
        throw e
      }
    },
    onColumnSelect() {
      const col = this.refColumns.find(c => this.relation.parentColumn === c.cn);
      this.$emit('onColumnSelect', col)
    }
  },

}
</script>

<style scoped>

.wrapper {
  border: solid 2px #7f828b33;
  border-radius: 4px;
}

/deep/ .v-input__append-inner {
  margin-top: 4px !important;
}
</style>
