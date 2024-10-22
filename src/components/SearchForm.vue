<template>
<div id="searchpanel" class="panel-body search-panel">
    <div class="row row-height">
        <div class="col-height col-md-3">
            <label for="usernames">{{ labels.usernames_label }}</label>
            <input type="text" v-model="localData.username" id="usernames" class="form-control input-md" maxlength="50" />
        </div>
        <div class="col-height col-md-3">
            <label for="emails">{{ labels.emails_label }}</label>
            <input type="text" v-model="localData.email" id="emails" class="form-control input-md" maxlength="100" />
        </div>
        <div class="col-height col-md-3">
            <label for="fromdates">{{ labels.fromdates_label }}</label>
            <InputDate v-model="localData.fromdate" id="fromdates" /> 
        </div>
        <div class="col-height col-md-3">
            <label for="todates">{{ labels.todates_label }}</label>
            <InputDate v-model="localData.todate" id="todates" /> 
        </div>
      </div>
      <div class="row row-height">
        <div class="col-height col-md-12 pull-right text-right">
          <button @click="searchClick" class="btn btn-dark btn-sm btn-ctrl"><i class="fa fa-search fa-btn-icon" aria-hidden="true"></i>{{ labels.search_button }}</button>
          <button @click="resetClick" class="btn btn-dark btn-sm btn-ctrl"><i class="fa fa-refresh fa-btn-icon" aria-hidden="true"></i>{{ labels.reset_button }}</button>
        </div>
    </div>
    <div id="listpanel" class="table-responsive fa-list-panel">
        <DataTable ref="dataTable" :settings="tableSettings" :labels="labels" :dataset="dataset" @data-select="dataSelected" @data-sort="dataSorted" :formater="formatData" />
        <DataPaging ref="dataPaging" :settings="pagingSettings" @page-select="pageSelected" />
    </div>
</div>
</template>
<script>
import { ref } from 'vue';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl, Utilities }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, serializeParameters }  from '@willsofts/will-app'
import { Paging } from "@willsofts/will-app";
import { InputDate, DataTable, DataPaging } from '@willsofts/will-control';

const curdate = Utilities.getDateNow();
const APP_URL = "/api/sfte017";
const defaultData = {
  username: "",
  email: "",
  fromdate: curdate,
  todate: curdate,
};

const tableSettings = {
    sequence: { label: "seqno_label" },
    columns: [
        {name: "username", type: "STRING", sorter: "username_surname", label: "username_surname_headerlabel" },
        {name: "email", type: "STRING", sorter: "email", label: "email_headerlabel" },
        {name: "issuer", type: "STRING", sorter: "issuer", label: "issuer_headerlabel" },
        {name: "createdate", type: "DATE", sorter: "createdatetime", label: "createdatetime_headerlabel", css: "text-center" },
        {name: "confirmdate", type: "DATE", sorter: "confirmdatetime", label: "confirmdatetime_headerlabel", css: "text-center" },
        {name: "factorflag", type: "STRING", sorter: "factorflag", label: "factorflag_headerlabel", css: "text-center", unescape: true },
    ],        
    actions: [
        {type: "button", action: "view"},
    ],
};

export default {
  components: {
    InputDate, DataTable, DataPaging
  },
  props: {
    labels: Object,
    formData: Object,
    dataCategory: Object
  },
  emits: ["data-select","data-insert"],
  setup(props) {
    const localData = ref({ ...defaultData, ...props.formData });
    let paging = new Paging();
    let pagingSettings = paging.setting;
    let filters = {};
    const dataset = ref({});
    return { localData, tableSettings, pagingSettings, paging, filters, dataset };
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
    insertClick() {
      this.$emit('data-insert',this.filters);
    },
    searchClick() {
      this.filters = {...this.localData};
      this.resetFilters();
      this.search();
    },
    resetFilters() {
      this.pagingSettings.page = 1;
      this.pagingSettings.orderBy = "";
      this.pagingSettings.orderDir = "";
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
          if(data.body) {
            this.$refs.dataTable.reset(data.body);
            this.$refs.dataPaging.reset(data.body?.offsets);
            this.pagingSettings.rows = data.body?.rows?.length;
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
      if(field.name=="factorflag") {
        if("1"==data) {
          return '<em class="fa fa-check-circle"></em>';
        } else {
          return '<em class="fa fa-circle-o"></em>';
        }
      } else if(field.name=="username") {
        return record.username+" "+record.usersurname;
      } else if(field.name=="createdate") {
        return data ? this.$refs.dataTable.formatField(data,field,record)+" "+record.createtime : "";
      } else if(field.name=="confirmdate") {
        return data ? this.$refs.dataTable.formatField(data,field,record)+" "+record.confirmtime : "";
      }
      return this.$refs.dataTable.formatField(data,field,record);
    },    
  }
};
</script>
