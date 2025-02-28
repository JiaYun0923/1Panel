<template>
    <div>
        <FireRouter />

        <div v-loading="loading">
            <FireStatus
                v-show="fireName !== '-'"
                ref="fireStatusRef"
                @search="search"
                v-model:loading="loading"
                v-model:name="fireName"
                v-model:mask-show="maskShow"
                v-model:status="fireStatus"
            />

            <div v-if="fireName !== '-'">
                <el-card v-if="fireStatus != 'running' && maskShow" class="mask-prompt">
                    <span>{{ $t('firewall.firewallNotStart') }}</span>
                </el-card>

                <LayoutContent :title="$t('firewall.ipRule')" :class="{ mask: fireStatus != 'running' }">
                    <template #toolbar>
                        <div class="flex justify-between gap-2 flex-wrap sm:flex-row">
                            <div class="flex flex-wrap gap-3">
                                <el-button type="primary" @click="onOpenDialog('create')">
                                    {{ $t('commons.button.create') }} {{ $t('firewall.ipRule') }}
                                </el-button>
                                <el-button @click="onDelete(null)" plain :disabled="selects.length === 0">
                                    {{ $t('commons.button.delete') }}
                                </el-button>
                            </div>
                            <div class="flex flex-wrap gap-3">
                                <TableSetting @search="search()" />
                                <TableSearch @search="search()" v-model:searchName="searchName" />
                            </div>
                        </div>
                    </template>
                    <template #search>
                        <div class="flx-align-center">
                            <el-select v-model="searchStrategy" @change="search()" clearable class="p-w-200">
                                <template #prefix>{{ $t('firewall.strategy') }}</template>
                                <el-option :label="$t('commons.table.all')" value=""></el-option>
                                <el-option :label="$t('firewall.allow')" value="accept"></el-option>
                                <el-option :label="$t('firewall.deny')" value="drop"></el-option>
                            </el-select>
                        </div>
                    </template>
                    <template #main>
                        <ComplexTable
                            :pagination-config="paginationConfig"
                            v-model:selects="selects"
                            @search="search"
                            :data="data"
                        >
                            <el-table-column type="selection" fix />
                            <el-table-column :min-width="80" :label="$t('firewall.address')" prop="address">
                                <template #default="{ row }">
                                    <span v-if="row.address && row.address !== 'Anywhere'">{{ row.address }}</span>
                                    <span v-else>{{ $t('firewall.allIP') }}</span>
                                </template>
                            </el-table-column>
                            <el-table-column :min-width="80" :label="$t('firewall.strategy')" prop="strategy">
                                <template #default="{ row }">
                                    <el-button
                                        v-if="row.strategy === 'accept'"
                                        @click="onChangeStatus(row, 'drop')"
                                        link
                                        type="success"
                                    >
                                        {{ $t('firewall.allow') }}
                                    </el-button>
                                    <el-button v-else link type="danger" @click="onChangeStatus(row, 'accept')">
                                        {{ $t('firewall.deny') }}
                                    </el-button>
                                </template>
                            </el-table-column>
                            <el-table-column
                                :min-width="150"
                                :label="$t('commons.table.description')"
                                prop="description"
                                show-overflow-tooltip
                            >
                                <template #default="{ row }">
                                    <fu-input-rw-switch v-model="row.description" @blur="onChange(row)" />
                                </template>
                            </el-table-column>
                            <fu-table-operations
                                width="200px"
                                :buttons="buttons"
                                :ellipsis="10"
                                :label="$t('commons.table.operate')"
                                fix
                            />
                        </ComplexTable>
                    </template>
                </LayoutContent>
            </div>
            <div v-else>
                <LayoutContent :title="$t('firewall.firewall')" :divider="true">
                    <template #main>
                        <div class="app-warn">
                            <div>
                                <span>{{ $t('firewall.notSupport') }}</span>
                                <span @click="toDoc">
                                    <el-icon class="ml-2"><Position /></el-icon>
                                    {{ $t('firewall.quickJump') }}
                                </span>
                                <div>
                                    <img src="@/assets/images/no_app.svg" />
                                </div>
                            </div>
                        </div>
                    </template>
                </LayoutContent>
            </div>
        </div>

        <OpDialog ref="opRef" @search="search" />
        <OperateDialog @search="search" ref="dialogRef" />
    </div>
</template>

<script lang="ts" setup>
import OperateDialog from '@/views/host/firewall/ip/operate/index.vue';
import FireRouter from '@/views/host/firewall/index.vue';
import FireStatus from '@/views/host/firewall/status/index.vue';
import { onMounted, reactive, ref } from 'vue';
import { batchOperateRule, searchFireRule, updateAddrRule, updateFirewallDescription } from '@/api/modules/host';
import { Host } from '@/api/interface/host';
import { ElMessageBox } from 'element-plus';
import i18n from '@/lang';
import { MsgSuccess } from '@/utils/message';

const loading = ref();
const activeTag = ref('address');
const selects = ref<any>([]);
const searchName = ref();
const searchStrategy = ref('');
const fireName = ref();

const maskShow = ref(true);
const fireStatus = ref('running');
const fireStatusRef = ref();

const opRef = ref();

const data = ref();
const paginationConfig = reactive({
    cacheSizeKey: 'firewall-ip-page-size',
    currentPage: 1,
    pageSize: 10,
    total: 0,
});

const search = async () => {
    if (fireStatus.value !== 'running') {
        loading.value = false;
        data.value = [];
        paginationConfig.total = 0;
        return;
    }
    let params = {
        type: activeTag.value,
        status: '',
        strategy: searchStrategy.value,
        info: searchName.value,
        page: paginationConfig.currentPage,
        pageSize: paginationConfig.pageSize,
    };
    loading.value = true;
    await searchFireRule(params)
        .then((res) => {
            loading.value = false;
            data.value = res.data.items || [];
            paginationConfig.total = res.data.total;
        })
        .catch(() => {
            loading.value = false;
        });
};

const dialogRef = ref();
const onOpenDialog = async (
    title: string,
    rowData: Partial<Host.RuleIP> = {
        strategy: 'accept',
    },
) => {
    let params = {
        title,
        rowData: { ...rowData },
    };
    dialogRef.value!.acceptParams(params);
};

const toDoc = () => {
    window.open('https://1panel.cn/docs/user_manual/hosts/firewall/', '_blank', 'noopener,noreferrer');
};

const onChange = async (info: any) => {
    info.type = 'address';
    await updateFirewallDescription(info);
    MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
};

const onChangeStatus = async (row: Host.RuleInfo, status: string) => {
    let operation =
        status === 'accept'
            ? i18n.global.t('firewall.changeStrategyIPHelper2')
            : i18n.global.t('firewall.changeStrategyIPHelper1');
    ElMessageBox.confirm(operation, i18n.global.t('firewall.changeStrategy', [' IP ']), {
        confirmButtonText: i18n.global.t('commons.button.confirm'),
        cancelButtonText: i18n.global.t('commons.button.cancel'),
    }).then(async () => {
        let params = {
            oldRule: {
                operation: 'remove',
                address: row.address,
                strategy: row.strategy,
                description: row.description,
            },
            newRule: {
                operation: 'add',
                address: row.address,
                strategy: status,
                description: row.description,
            },
        };
        loading.value = true;
        await updateAddrRule(params)
            .then(() => {
                loading.value = false;
                MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
                search();
            })
            .catch(() => {
                loading.value = false;
            });
    });
};

const onDelete = async (row: Host.RuleIP | null) => {
    let names = [];
    let rules = [];
    if (row) {
        rules.push({
            operation: 'remove',
            address: row.address,
            port: '',
            source: '',
            protocol: '',
            strategy: row.strategy,
        });
        names = [row.address];
    } else {
        for (const item of selects.value) {
            rules.push({
                operation: 'remove',
                address: item.address,
                port: '',
                source: '',
                protocol: '',
                strategy: item.strategy,
            });
            names.push(item.address);
        }
    }
    opRef.value.acceptParams({
        title: i18n.global.t('commons.button.delete'),
        names: names,
        msg: i18n.global.t('commons.msg.operatorHelper', [
            i18n.global.t('firewall.ipRule'),
            i18n.global.t('commons.button.delete'),
        ]),
        api: batchOperateRule,
        params: { type: 'address', rules: rules },
    });
};

const buttons = [
    {
        label: i18n.global.t('commons.button.edit'),
        click: (row: Host.RuleIP) => {
            onOpenDialog('edit', row);
        },
    },
    {
        label: i18n.global.t('commons.button.delete'),
        click: (row: Host.RuleIP) => {
            onDelete(row);
        },
    },
];

onMounted(() => {
    if (fireName.value !== '-') {
        loading.value = true;
        fireStatusRef.value.acceptParams();
    }
});
</script>
