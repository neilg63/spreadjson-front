<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import ToolingIcon from './icons/IconTooling.vue'
import EcosystemIcon from './icons/IconEcosystem.vue'
import CommunityIcon from './icons/IconCommunity.vue'
import SupportIcon from './icons/IconSupport.vue'
import FileUpload from 'primevue/fileupload'

import type { FileUploadUploadEvent } from 'primevue/fileupload'
import { Button, Column, DataTable, InputNumber, InputText } from 'primevue'

const valid_format_string = '.xlsx,.xlsb,.xls,.ods,.csv,.tsv,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet,application/vnd.ms-excel,application/vnd.oasis.opendocument.spreadsheet,text/csv,text/tab-separated-values';

const max_file_size = 20 * 1000 * 1000;

interface FieldValue {
  [field:string]: any;
}


const fileName = ref<string>("");

const loaded = ref<boolean>(false);

const spreadData = ref<FieldValue[]>([]);

const headerKeys = ref<string[]>([]);

const editedHeaderKeys = ref<string[]>([]);

const maxRows = ref<number>(1000);

const numRows = ref<number>(0);

const hasData = computed(() => spreadData.value.length > 0);
const hasFileName = computed(() => fileName.value !== '');


async function upload(event: any) {
  const formData = new FormData();
  loaded.value = false;
  const files = Array.isArray(event.files) ? event.files : [event.files];
  for (let file of files) {
    formData.append('file', file);
    break;
  }

  formData.append('max', maxRows.value.toString());
  try {
    const response = await fetch('http://127.0.0.1:3000/', {
      method: 'POST',
      body: formData
    });

    if (!response.ok) {
      throw new Error('Network response was not ok');
    }

    const result = await response.json();
    if (result instanceof Object) {
      const { data, fields, name, num_rows } = result;
      if (data instanceof Array) {
        spreadData.value = data.filter(row => row instanceof Object).map((row, index) => {
          row._num = index + 1;
          return row;
        });
        setTimeout(() => {
          loaded.value = true;
          storeCurrentData();
        }, 500);
      }
      if (fields instanceof Array) {
        headerKeys.value = fields;
        editedHeaderKeys.value = [...fields];
      }
      if (name) {
        fileName.value = name;
      }
      numRows.value = typeof num_rows === 'number' ? num_rows : 0;
    } else {
      console.error('Error:', result);
    }
  } catch (error) {
    console.error('Error:', error);
  }
}

function handleRowCellClick(target: HTMLElement) {
  const par = target.parentElement;
  console.log('Row cell clicked:', par?.getAttribute('data-p-index'));
}


function handleTableClick(event: PointerEvent) {
  if (!(event.target instanceof HTMLElement)) {
    return;
  }
  const par = event.target.parentElement;
  const tgName = event.target.tagName.toLowerCase();
  const useParent = par instanceof HTMLElement && tgName === 'div';
  const refTgName = useParent ? par?.tagName.toLocaleLowerCase() : tgName;
  const refEl = useParent ? par : event.target;
  switch (refTgName) {
    case 'td':
      handleRowCellClick(event.target);
      break;
  }
}

function assignRowClass(row: FieldValue) {
  const { _num } = row;
  const cls = _num % 2 === 0 ? 'row-even' : 'row-odd';
  return [cls, ['row-' + _num]];
}

function updateHeaderKey(event: any, index: number) {
  if (index < 0 || index >= editedHeaderKeys.value.length) {
    return;
  }
  const newVal = toSnakeCase(event.target.value);
  if (newVal.length > 0 ) {
    editedHeaderKeys.value[index] = newVal;
  }
}

function simplifyIntlLetters(str: string): string {
  const replMap = [
    ['a', 'å', 'á', 'à', 'ä', 'ã', 'â'],
    ['e', 'é', 'è', 'ë', 'ê'],
    ['i', 'í', 'ì', 'ï', 'î'],
    ['o', 'ó', 'ò', 'ö', 'ô', 'õ'],
    ['u', 'ú', 'ù', 'ü'],
    ['ny', 'ñ'],
    ['ss', 'ß'],
    ['s', 'ś', 'ŝ', 'Ş', 'ṣ'],
    ['c', 'ç', 'ć', 'č'],
    ['d', 'ð'],
  ];
  const letters = replMap.reduce((a, b) => a.concat(b.slice(1)), []);
  const allLetters = new RegExp('[^a-z0-9' + letters.join('') + ']', 'i');
  str = str.trim().replace(allLetters, '.*?');
  replMap.forEach(row => {
    if (row.length > 1) {
      const sourceSet = row.slice(1).join('');
      const sourcePat = new RegExp(`[${sourceSet}]`, 'gi');
      const repl = row[0];
      str = str.replace(sourcePat, repl);
    }
  });
  return str;
};


function toSnakeCase(str: string): string {
  return simplifyIntlLetters(str).trim().replace(/([^a-z0-9]+)/gi, '_');
}



function storeData(record: any) {
  if (record instanceof Object) {
    const saved = window.localStorage.setItem('currentData', JSON.stringify(record));
  }
}

function storeCurrentData() {
  const data = spreadData.value || [];
  const saveData = {
    data,
    fields: editedHeaderKeys.value,
    fileName: fileName.value,
    num_rows: numRows.value
  };
  storeData(saveData);
}

function loadCurrentData() {
  const record = restoreData('currentData');
  if (record instanceof Object) {
    spreadData.value = record.data;
    headerKeys.value = record.fields;
    editedHeaderKeys.value = [...record.fields];
    fileName.value = record.fileName;
    numRows.value = record.num_rows;
    setTimeout(() => {
      loaded.value = true;
    }, 500);
  }
}
function restoreData(key: string) {
  const saved = window.localStorage.getItem(key);
  if (typeof saved === 'string') {
    const data = JSON.parse(saved);
    if (data instanceof Object) {
      return data;
    }
  }
}

function downloadAsJson() {
  if (!hasData.value) {
    return;
  }
  const data = spreadData.value.map(row => {
    const newRow: Map<string, any> = new Map();
    for (let i = 0; i < headerKeys.value.length; i++) {
      const key = headerKeys.value[i];
      const editedKey = i < editedHeaderKeys.value.length ? editedHeaderKeys.value[i] : key;
      const refKey = editedKey.length > 0 ? editedKey : key;
      newRow.set(refKey, row[key]);
    }
    return Object.fromEntries(newRow.entries());
  });
  const jsonStr = JSON.stringify(data, null, 2);
  const blob = new Blob([jsonStr], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `${fileName.value || 'data'}.json`;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url); 
}

// Call loadCurrentData in the onMounted lifecycle hook
onMounted(() => {
  loadCurrentData();
});

</script>

<template>
  <section class="spreadsheet-main">
    <h2>Upload your Spreadsheets to create an API endpoint</h2>
    <FileUpload name="file" @uploader="upload" :customUpload="true" :multiple="false" :accept="valid_format_string" :maxFileSize="max_file_size">
      <template #empty>
          <span>Drag and drop files to here to upload.</span>
      </template>
    </FileUpload>
    <InputNumber v-model="maxRows" :min="5" :max="100000" :step="1" />
    
    <dl v-if="hasData" class="data-info grid-2">
      <dt v-if="hasFileName">File name</dt>
      <dd v-if="hasFileName">{{ fileName }}</dd>
      <dt>Rows</dt>
      <dd>{{ spreadData.length }} / {{ numRows }}</dd>
      <dt>Columns</dt>
      <dd>{{ headerKeys.length }}</dd>
    </dl>
    <div v-if="hasData" class="flex flex-row">
      <div v-for="(hk, index) in editedHeaderKeys" class="header-col">
        <InputText type="text" :value="hk" @input="updateHeaderKey($event, index)" />
      </div>
    </div>
    <Button label="Download as JSON" icon="pi pi-download" iconPos="left" @click="downloadAsJson" />
    <DataTable v-if="hasData" :value="spreadData" :showGridlines="true" tableStyle="min-width: 50rem; width: 100%;" @click="handleTableClick" :row-hover="true" :row-class="assignRowClass">
      <Column field="_num" header="#" />
      <Column v-for="(hk, index) in headerKeys" :key="index" :field="hk" :header="hk" />
    </DataTable>
  </section>

  
</template>

<style lang="css">

dl.data-info {
  display: grid;
  grid-template-columns: 10rem auto;
}

.flex-row {
  display: flex;
  flex-direction: row;
}


</style>