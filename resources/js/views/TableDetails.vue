<template>
    <div class="flex-grow flex flex-col overflow-x-auto soft-scroll">
        <div class="flex flex-col h-0 flex-1">
            <div class="bg-white w-full">
                <div class="px-12">
                    <div class="py-6">
                        <div class="flex items-center justify-between text-sm text-gray-700 uppercase font-bold tracking-widest">
                            <div>Table {{ tableName }}</div>
                            <div class="flex">
                                <button
                                    class="inline-flex items-center px-4 py-2 border rounded-md font-semibold text-xs uppercase tracking-widest transition ease-in-out duration-150 rounded-r-none"
                                    :class="tab == 'data' ? 'bg-gray-800 border-transparent text-white hover:bg-gray-700 active:bg-gray-900 focus:outline-none focus:border-gray-900 focus:shadow-outline-gray disabled:opacity-75' : 'bg-white border-gray-300 text-gray-700 shadow-sm hover:text-gray-500 focus:outline-none focus:border-blue-300 focus:shadow-outline-blue active:text-gray-800 active:bg-gray-50'"
                                    :disabled="tab == 'data'"
                                    @click="tab = 'data'"
                                >
                                    Data
                                </button>
                                <button
                                    class="inline-flex items-center px-4 py-2 border rounded-md font-semibold text-xs uppercase tracking-widest transition ease-in-out duration-150 rounded-l-none rounded-r-none"
                                    :class="tab == 'structure' ? 'bg-gray-800 border-transparent text-white hover:bg-gray-700 active:bg-gray-900 focus:outline-none focus:border-gray-900 focus:shadow-outline-gray disabled:opacity-75' : 'bg-white border-gray-300 text-gray-700 shadow-sm hover:text-gray-500 focus:outline-none focus:border-blue-300 focus:shadow-outline-blue active:text-gray-800 active:bg-gray-50'"
                                    :disabled="tab == 'structure'"
                                    @click="tab = 'structure'"
                                >
                                    Structure
                                </button>
                                <button
                                    class="inline-flex items-center px-4 py-2 border rounded-md font-semibold text-xs uppercase tracking-widest transition ease-in-out duration-150 rounded-l-none"
                                    :class="tab == 'info' ? 'bg-gray-800 border-transparent text-white hover:bg-gray-700 active:bg-gray-900 focus:outline-none focus:border-gray-900 focus:shadow-outline-gray disabled:opacity-75' : 'bg-white border-gray-300 text-gray-700 shadow-sm hover:text-gray-500 focus:outline-none focus:border-blue-300 focus:shadow-outline-blue active:text-gray-800 active:bg-gray-50'"
                                    :disabled="tab == 'info'"
                                    @click="tab = 'info'"
                                >
                                    Info
                                </button>
                            </div>
                        </div>

                        <div class="mt-6 border-t-2 border-gray-200" v-if="tab == 'data' && filterEnabled">
                            <div class="flex flex-col w-full">
                                <form>
                                    <div class="flex mt-4">
                                        <select class="py-2 px-3 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm pr-8" v-model="filterForm.field">
                                            <option v-for="column in tableColumns" :key="`column-name-${column.column_name}`" :value="column.column_name">
                                                {{ column.column_name }}
                                            </option>
                                            <hr>
                                            <option value="__any__">Any column</option>
                                            <option value="__raw__">Raw SQL</option>
                                        </select>
                                        <select class="py-2 px-3 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm pr-8 ml-2" v-model="filterForm.operator" v-show="filterForm.field != '__raw__'">
                                            <option value="=">=</option>
                                            <option value="<>">&lt;&gt;</option>
                                            <option value="<">&lt;</option>
                                            <option value=">">&gt;</option>
                                            <option value="<=">&lt;=</option>
                                            <option value=">=">&gt;=</option>
                                            <hr>
                                            <option value="IN">IN</option>
                                            <option value="NOT IN">NOT IN</option>
                                            <hr>
                                            <option value="IS NULL">IS NULL</option>
                                            <option value="IS NOT NULL">IS NOT NULL</option>
                                            <hr>
                                            <option value="BETWEEN">BETWEEN</option>
                                            <option value="NOT BETWEEN">NOT BETWEEN</option>
                                            <hr>
                                            <option value="LIKE">LIKE</option>
                                            <option value="NOT LIKE">NOT LIKE</option>
                                        </select>
                                        <x-input class="flex-1 block w-full ml-2 px-4 py-2" :placeholder="filterValuePlaceholder" v-model="filterForm.value" :disabled="isNullOrNotNullOperator" />
                                        <x-button class="ml-2" @click.native="loadEntries" :disabled="loadingEntries">Apply</x-button>
                                    </div>
                                </form>
                            </div>
                        </div>
                    </div>
                </div>
            </div>

            <template v-if="! ready">
                <div class="flex-1 flex flex-col">
                    <x-loader />
                </div>
            </template>

            <template v-else>
                <template v-if="tab == 'data'">
                    <div class="flex-1 flex flex-col overflow-y-auto bg-gray-200">
                        <div class="block min-w-full overflow-x-auto soft-scroll">
                            <table class="divide-y divide-gray-200 text-gray-800">
                                <thead class="bg-gray-50">
                                    <tr>
                                        <th
                                            v-for="column in tableColumns"
                                            :key="column.column_name"
                                            class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider cursor-pointer"
                                            @click.prevent="updateSorting(column.column_name)"
                                        >
                                            <div class="flex justify-between items-center">
                                                <span></span>
                                                <span>{{ column.column_name }}</span>
                                                <span>
                                                    <span v-if="column.column_name == sortKey">
                                                        <svg xmlns="http://www.w3.org/2000/svg" class="h-3 w-3" fill="none" viewBox="0 0 24 24" stroke="currentColor" v-if="sortDir == 'asc'">
                                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 10l7-7m0 0l7 7m-7-7v18" />
                                                        </svg>
                                                        <svg xmlns="http://www.w3.org/2000/svg" class="h-3 w-3" fill="none" viewBox="0 0 24 24" stroke="currentColor" v-else>
                                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 14l-7 7m0 0l-7-7m7 7V3" />
                                                        </svg>
                                                    </span>
                                                </span>
                                            </div>
                                        </th>
                                    </tr>
                                </thead>

                                <tbody class="bg-white divide-y divide-gray-200">
                                    <tr v-for="(entry, index) in entries" :key="index">
                                        <td class="px-6 py-4 whitespace-nowrap text-sm" v-for="column in tableColumns" :key="column.column_name" v-on:click.prevent="openDetailEntry(entry)">
                                            <x-field-value :value="entry[column.column_name] == null ? entry[column.column_name] : strLimit(String(entry[column.column_name]))" />
                                        </td>
                                    </tr>
                                </tbody>
                            </table>
                        </div>
                    </div>

                    <div class="bg-gray-100 border-t-2">
                        <div class="px-12">
                            <div class="py-4">
                                <div class="flex justify-between items-center">
                                    <div class="flex">
                                        <button
                                            class="inline-flex items-center px-4 py-2 border rounded-md font-semibold text-xs uppercase tracking-widest transition ease-in-out duration-150"
                                            :class="filterEnabled ? 'bg-gray-800 border-transparent text-white hover:bg-gray-700 active:bg-gray-900 focus:outline-none focus:border-gray-900 focus:shadow-outline-gray disabled:opacity-75' : 'bg-white border-gray-300 text-gray-700 shadow-sm hover:text-gray-500 focus:outline-none focus:border-blue-300 focus:shadow-outline-blue active:text-gray-800 active:bg-gray-50'"
                                            @click="toggleFilter"
                                        >
                                            Filters
                                        </button>
                                    </div>
                                    <div class="text-sm text-gray-700">
                                        <span v-if="loadingEntries">Loading rows...</span>
                                        <span v-else>
                                            <span v-if="total > 0">
                                                {{ formatNumber(from) }}-{{ formatNumber(to) }} of {{ formatNumber(total) }} rows
                                            </span>
                                            <span v-else>
                                                0 rows
                                            </span>
                                        </span>
                                    </div>
                                    <div class="flex">
                                        <button
                                            class="inline-flex items-center px-4 py-2 border border-gray-300 text-sm leading-5 font-medium rounded-md text-gray-700 bg-white hover:text-gray-500 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 active:text-gray-800 active:bg-gray-50 transition duration-150 ease-in-out rounded-r-none disabled:opacity-25"
                                            title="Previous page"
                                            :disabled="offset == 0 || loadingEntries"
                                            @click.prevent="selectPreviousPage"
                                        >
                                            <icon-chevron-left size="4"></icon-chevron-left>
                                        </button>
                                        <button
                                            class="inline-flex items-center px-4 py-2 border border-gray-300 text-sm leading-5 font-medium rounded-md text-gray-700 bg-white hover:text-gray-500 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 active:text-gray-800 active:bg-gray-50 transition duration-150 ease-in-out rounded-l-none rounded-r-none"
                                            title="Page settings"
                                            :disabled="loadingEntries"
                                            @click.prevent="startSetting"
                                        >
                                            <icon-cog size="4"></icon-cog>
                                        </button>
                                        <button
                                            class="inline-flex items-center px-4 py-2 border border-gray-300 text-sm leading-5 font-medium rounded-md text-gray-700 bg-white hover:text-gray-500 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 active:text-gray-800 active:bg-gray-50 transition duration-150 ease-in-out rounded-l-none disabled:opacity-25"
                                            title="Next page"
                                            :disabled="entries.length < limit || loadingEntries"
                                            @click.prevent="selectNextPage"
                                        >
                                            <icon-chevron-right size="4"></icon-chevron-right>
                                        </button>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </template>

                <template v-if="tab == 'structure'">
                    <div class="block min-w-full overflow-x-auto soft-scroll">
                        <table class="divide-y divide-gray-200 text-gray-800">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">#</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Column Name</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Data Type</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Collation</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Is Nullable</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Key</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Column Default</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Extra</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Comment</th>
                                </tr>
                            </thead>

                            <tbody class="bg-white divide-y divide-gray-200">
                                <tr v-for="(column, index) in tableColumns" :key="index">
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="index + 1" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.column_name" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.data_type" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.collation" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">{{ column.is_nullable ? 'YES' : 'NO' }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.key" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.column_default" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.extra" />
                                    </td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">
                                        <x-field-value :value="column.comment" />
                                    </td>
                                </tr>
                            </tbody>
                        </table>
                    </div>

                    <div class="block min-w-full overflow-x-auto soft-scroll mt-4">
                        <table class="divide-y divide-gray-200 text-gray-800">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Index Name</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Index Algorithm</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Is Unique</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider">Column Name</th>
                                </tr>
                            </thead>

                            <tbody class="bg-white divide-y divide-gray-200">
                                <tr v-for="index in tableIndexes" :key="index.index_name">
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">{{ index.index_name }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">{{ index.index_algorithm }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">{{ index.is_unique }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap text-sm">{{ index.column_name }}</td>
                                </tr>
                            </tbody>
                        </table>
                    </div>
                </template>

                <template v-if="tab == 'info'">
                    <table class="min-w-full divide-y divide-gray-200 text-gray-800">
                        <tbody class="bg-white divide-y divide-gray-200">
                            <tr v-for="row in tableInfo" :key="row.field">
                                <th scope="row" class="px-6 py-3 text-left text-xs font-semibold uppercase tracking-wider bg-gray-50 w-1/4">{{ row.field }}</th>
                                <td class="px-6 py-4 whitespace-nowrap text-sm">
                                    <x-field-value :value="row.value" />
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </template>
            </template>
        </div>

        <x-dialog-modal :show="setting" maxWidth="sm">
            <template #title>
                Page settings
            </template>
            <template #content>
                <form @submit.prevent="setPageSettings">
                    <div>
                        <x-label for="limit" value="Limit" />
                        <x-input id="limit" type="text" class="mt-1 block w-full" placecholder="Limit" v-model="pageSettingsForm.limit" required autofocus />
                    </div>
                    <div class="mt-4">
                        <x-label for="offset" value="Offset" />
                        <x-input id="offset" type="text" class="mt-1 block w-full" placecholder="Offset" v-model="pageSettingsForm.offset" required />
                    </div>
                </form>
            </template>
            <template #footer>
                <x-secondary-button @click.native="closeModal">
                    Nevermind
                </x-secondary-button>

                <x-button class="ml-2" @click.native="setPageSettings" :class="{ 'opacity-25': loadingEntries }" :disabled="loadingEntries">
                    Go
                </x-button>
            </template>
        </x-dialog-modal>

        <x-dialog-modal :show="showEntryDetail" max-width="7xl">
            <template #title>
                Entry detail
            </template>
            <template #content>
                <div class="flex flex-col divide-y">
                    <div v-for="(data, key) in detailEntry" :key="`entry-data-${key}`" class="flex">
                        <div class="w-1/4 py-4 px-2">
                            <h4 class="font-bold">{{ key }}</h4>
                        </div>
                        <div class="w-3/4 py-4 px-2 break-words">
                            {{ data }}
                        </div>
                    </div>
                </div>
            </template>
            <template #footer>
                <x-secondary-button @click.native="closeDetailEntry">
                    Close
                </x-secondary-button>
            </template>
        </x-dialog-modal>
    </div>
</template>

<script>
export default {
    props: ['tableName'],

    data() {
        return {
            tab: 'data',
            tableInfo: [],
            tableColumns: [],
            tableIndexes: [],
            entries: [],
            total: 0,
            offset: 0,
            limit: 50,
            sortKey: null,
            sortDir: 'asc',
            ready: false,
            loadingEntries: true,
            setting: false,
            showEntryDetail: false,
            detailEntry: {},
            pageSettingsForm: {
                offset: null,
                limit: null,
            },
            filterEnabled: false,
            filterForm: {
                field: '__raw__',
                operator: '=',
                value: '',
            },
            filterValuePlaceholder: 'EMPTY',
        };
    },

    computed: {
        from() {
            return this.offset + 1;
        },

        to() {
            return this.offset + this.entries.length;
        },

        isNullOrNotNullOperator() {
            return ['IS NULL', 'IS NOT NULL'].includes(this.filterForm.operator);
        },
    },

    watch: {
        tableName: function () {
            this.filterEnabled = false;
            this.filterForm = {
                field: '__raw__',
                operator: '=',
                value: '',
            };
            this.entries = [];
            this.offset = 0;
            this.sortKey = null;
            this.sortDir = 'asc';
            this.loadTable();
            this.loadEntries();
        },
        'filterForm.operator': function (newVal) {
            if (['IN', 'NOT IN'].includes(newVal)) {
                this.filterValuePlaceholder = '1,2,3';
            } else if (['IS NULL', 'IS NOT NULL'].includes(newVal)) {
                this.filterForm.value = '';
                this.filterValuePlaceholder = '';
            } else if (['BETWEEN', 'NOT BETWEEN'].includes(newVal)) {
                this.filterValuePlaceholder = '1 AND 100';
            } else if (['LIKE', 'NOT LIKE'].includes(newVal)) {
                this.filterValuePlaceholder = 'Pattern';
            } else {
                this.filterValuePlaceholder = 'EMPTY';
            }
        },
    },

    mounted() {
        this.loadTable();
        this.loadEntries();
    },

    methods: {
        async loadTable() {
            this.ready = false;

            const { data } = await axios.get(`${Dibi.path}/api/tables/${this.tableName}`);

            this.tableInfo = _.map(Object.keys(data.info), key => {
                if (key === 'raw') {
                    return;
                }

                if (['rows', 'avgRowLength', 'autoIncrement'].includes(key)) {
                    return {
                        field: _.startCase(key),
                        value: this.formatNumber(data.info[key]),
                    };
                }

                if (['dataLength', 'dataFree'].includes(key)) {
                    return {
                        field: _.startCase(key),
                        value: this.formatNumber(data.info[key], '0.0 ib'),
                    };
                }

                if (['maxDataLength', 'indexLength'].includes(key)) {
                    return {
                        field: _.startCase(key),
                        value: this.formatNumber(data.info[key], '0 b'),
                    };
                }

                return {
                    field: _.startCase(key),
                    value: data.info[key],
                };
            }).filter(value => value);

            this.tableColumns = data.columns;
            this.tableIndexes = data.indexes;
            this.ready = true;
        },

        async loadEntries() {
            this.loadingEntries = true;

            const url = `${Dibi.path}/api/tables/${this.tableName}/rows?offset=${this.offset}&limit=${this.limit}&sort_key=${this.sortKey ? this.sortKey : ''}&sort_dir=${this.sortDir}`;

            const data = this.filterEnabled ? {
                filters: [
                    this.filterForm,
                ],
            } : {};

            try {
                const response = await axios.post(
                    `${Dibi.path}/api/tables/${this.tableName}/rows/filter?offset=${this.offset}&limit=${this.limit}&sort_key=${this.sortKey ? this.sortKey : ''}&sort_dir=${this.sortDir}`,
                    data
                );

                this.entries = response.data.data;
                this.total = response.data.total;
                this.loadingEntries = false;
            } catch (error) {
                this.loadingEntries = false;
            }
        },

        updateSorting(sortKey) {
            if (this.sortKey == sortKey) {
                if (this.sortDir == 'asc') {
                    this.sortDir = 'desc';
                } else {
                    this.sortDir = 'asc';
                }
            } else {
                this.sortDir = 'asc';
            }

            this.sortKey = sortKey;

            this.loadEntries();
        },

        selectPreviousPage() {
            if (this.offset > this.limit) {
                this.offset = this.offset - this.limit;
            } else {
                this.offset = 0;
            }

            this.loadEntries();
        },

        selectNextPage() {
            this.offset = this.offset + this.limit;

            this.loadEntries();
        },

        startSetting() {
            this.pageSettingsForm = {
                limit: this.limit,
                offset: this.offset,
            };
            this.setting = true;
        },

        setPageSettings() {
            this.limit = parseInt(this.pageSettingsForm.limit);
            this.offset = parseInt(this.pageSettingsForm.offset);
            this.loadEntries();
            this.closeModal();
        },

        closeModal() {
            this.setting = false;
        },

        toggleFilter() {
            this.filterEnabled = !this.filterEnabled;

            if (! this.filterEnabled) {
                this.loadEntries();
            }
        },

        openDetailEntry(entry) {
            console.log(entry);
            this.detailEntry = entry;
            this.showEntryDetail = true;
        },

        closeDetailEntry() {
            this.showEntryDetail = false;
        }
    },
};
</script>
