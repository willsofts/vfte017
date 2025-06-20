<!-- App.vue -->
<template>
  <div id="fswaitlayer" class="fa fa-spinner fa-spin"></div>
  <div class="pt-page pt-page-current pt-page-controller search-pager">
    <PageHeader ref="pageHeader" :labels="labels" pid="vfte017" version="1.0.0" showLanguage="true" @language-changed="changeLanguage" :multiLanguages="multiLanguages" :build="buildVersion" :visible="displayPageHeader" />
    <SearchForm ref="searchForm" :labels="labels" :dataCategory="dataCategory" @data-select="dataSelected" />
  </div>
  <teleport to="#modaldialog">
    <EntryForm ref="entryForm" :labels="labels" :dataCategory="dataCategory" @data-deleted="dataDeleted" />
  </teleport>
</template>
<script>
import { ref } from 'vue';
import { PageHeader } from '@willsofts/will-control';
import SearchForm from '@/components/SearchForm.vue';
import EntryForm from '@/components/EntryForm.vue';
import { getLabelModel, getMultiLanguagesModel, getMetaInfo } from "@willsofts/will-app";
import { getDefaultLanguage, setDefaultLanguage } from "@willsofts/will-app";
import { startApplication } from "@willsofts/will-app";

const buildVersion = process.env.VUE_APP_BUILD_DATETIME;
export default {
  components: {
    PageHeader, SearchForm, EntryForm
  },
  setup() {
    const dataCategory = { };
    let labels = ref(getLabelModel());
    const multiLanguages = ref(getMultiLanguagesModel());
	const displayPageHeader = !(String(getMetaInfo().DISPLAY_PAGE_HEADER)=="false");
    return { displayPageHeader, buildVersion, multiLanguages, labels, dataCategory };
  },
  mounted() {
    console.log("App: mounted ...");
    this.$nextTick(async () => {
      //ensure ui completed then invoke startApplication 
      startApplication("vfte017",(data) => {
        console.log("vueapp: message",data);
        if(data.type=="language") {
          let lang = data.language;
          if(lang) {
            this.changeLanguage(lang);
          }
        } else {
          this.multiLanguages = getMultiLanguagesModel();
          this.messagingHandler(data);
          this.$refs.pageHeader.changeLanguage(getDefaultLanguage());
        }
      });
    });
  },
  methods: {
    messagingHandler(data) {
      console.log("messagingHandler: data",data); 
    },
    changeLanguage(lang) {
      setDefaultLanguage(lang);
      let labelModel = getLabelModel(lang);
      this.labels = labelModel;
    },
    dataSelected(item,action) {
      //listen action from search form
      console.log("App: dataSelected",item,"action",action);
      if("edit"==action || "view"==action) {
        this.$refs.entryForm.retrieveRecord({userid: item.userid});
      } else if("delete"==action) {
        this.$refs.entryForm.startDeleteRecord({userid: item.userid});
      }
    },
    dataDeleted(data,response) {
      //listen action from entry form when after deleted
      console.log("App: record deleted");
      console.log("data",data,"response",response);
      this.$refs.searchForm.search(true);
    },
  }
};
</script>