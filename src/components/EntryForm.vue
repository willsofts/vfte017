<template>
  <DialogForm ref="dialogForm">
    <template v-slot:header>
      <h4 class="modal-title">{{ labels.modalheadertitle_show }}</h4>
    </template>
    <template v-slot:default>
        <div class="row row-height">
          <div class="col-height col-md-8">
            <span><label>{{ labels.userid_label }}</label> : {{ localData.userfullname }}</span>
          </div>
        </div>
        <div id="listpanelhistory" class="table-responsive fa-list-panel">
          <DataTable ref="dataTable" :settings="tableSettings" :labels="labels" :dataset="dataset" @data-select="dataSelected" @data-sort="dataSorted" :formater="formatData" />
          <DataPaging ref="dataPaging" :settings="pagingSettings" @page-select="pageSelected" />          
        </div>
    </template>
    <template v-slot:footer>
      <button class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
    </template>
  </DialogForm>
</template>
<script>
import { ref } from 'vue';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure }  from '@willsofts/will-app';
import { serializeParameters } from '@willsofts/will-app';
import { Paging } from "@willsofts/will-app";
import { DataTable, DataPaging } from '@willsofts/will-control';
import DialogForm from './DialogForm.vue';

const APP_URL = "/api/sfte017history";
const defaultData = {
  userid: "",
  userfullname: "",
};

const tableSettings = {
    sequence: { label: "seqno_label" },
    columns: [
        {name: "email", type: "STRING", sorter: "email", label: "email_headerlabel" },
        {name: "issuer", type: "STRING", sorter: "issuer", label: "issuer_headerlabel" },
        {name: "createdate", type: "DATE", sorter: "createdatetime", label: "createdatetime_headerlabel", css: "text-center" },
        {name: "confirmdate", type: "DATE", sorter: "confirmdatetime", label: "confirmdatetime_headerlabel", css: "text-center" },
    ],        
};

export default {
  components: {
    DialogForm, DataTable, DataPaging 
  },
  props: {
    modes: Object,
    labels: Object,
    dataCategory: Object
  },
  emits: ["data-saved","data-updated","data-deleted"],
  setup() {
    const localData = ref({ ...defaultData }); 
    let paging = new Paging();
    let pagingSettings = paging.setting;
    let filters = {};
    const dataset = ref({});
    return { localData, tableSettings, pagingSettings, paging, filters, dataset };
  },
  computed: {
  },
  mounted() {
    this.$nextTick(function () {
      $("#modaldialog_layer").find(".modal-dialog").draggable();
    });
  },
  methods: {
    reset(newData) {
      if(newData) this.localData = {...newData};
    },
    getPagingOptions(settings) {
      if(!settings) settings = this.pagingSettings;
      return {page: settings.page, limit: settings.limit, rowsPerPage: settings.rowsPerPage, orderBy: settings.orderBy?settings.orderBy:"", orderDir: settings.orderDir?settings.orderDir:"" };
    },
    resetClick() {
      this.localData = {...defaultData};
      this.resetFilters();
      this.$refs.dataTable.clear();
      this.$refs.dataPaging.clear();
      this.pagingSettings.rows = 0;
    },
    resetFilters() {
      this.pagingSettings.page = 1;
      this.pagingSettings.orderBy = "";
      this.pagingSettings.orderDir = "";
    },
    showDialog() {
      $(this.$refs.dialogForm.$el).modal("show");
    },  
    hideDialog() {
      $(this.$refs.dialogForm.$el).modal("hide");
    },
    retrieveRecord(data) {
      this.reset(data);
      this.filters = {...data};
      this.resetFilters();
      this.search();
    },    
    search(ensurePaging=false) {
      if(ensurePaging) {
        if(this.pagingSettings.rows <= 1 && this.pagingSettings.page > 1) {
          this.pagingSettings.page = this.pagingSettings.page - 1;
        }
      }
      console.log("search: pagingSettings",this.pagingSettings);
      let options = this.getPagingOptions();
      this.collecting(options,this.filters);
    },
    collecting(options,criterias) {
      console.log("collecting: options",options,", criteria",criterias);
      let jsondata = Object.assign({ajax: true},options);
      let formdata = serializeParameters(jsondata,criterias);
      startWaiting();
      $.ajax({
        url: getApiUrl()+APP_URL+"/collect",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("error: status",status,"errorThrown",errorThrown);
          submitFailure(transport,status,errorThrown);
        },
        success: (data) => {
          stopWaiting();
          console.log("collecting: success",data);
          if(data.body?.dataset) {
            this.reset(data.body.meta);
            this.$refs.dataTable.reset(data.body.dataset);
            this.$refs.dataPaging.reset(data.body.dataset?.offsets);
            this.pagingSettings.rows = data.body.dataset?.rows?.length;
            this.showDialog();
          }
        }
      });	
    },
    pageSelected(item) {
      console.log("page selected:",item);
      this.pagingSettings.page = item.page;
      let options = this.getPagingOptions();
      this.collecting(options,this.filters);
    },
    dataSelected(item,action) {
      console.log("dataSelected",item,"action",action);
      this.$emit('data-select', item, action);
    },
    dataSorted(sorter,direction) {
      console.log("dataSorted",sorter,"direction",direction);
      this.pagingSettings.orderBy = sorter;
      this.pagingSettings.orderDir = direction;
      let options = this.getPagingOptions();
      this.collecting(options,this.filters);
    },
    formatData(data,field,record) {
      if(field.name=="createdate") {
        return data ? this.$refs.dataTable.formatField(data,field,record)+" "+record.createtime : "";
      } else if(field.name=="confirmdate") {
        return data ? this.$refs.dataTable.formatField(data,field,record)+" "+record.confirmtime : "";
      }
      return this.$refs.dataTable.formatField(data,field,record);
    },    
  }
};
</script>
