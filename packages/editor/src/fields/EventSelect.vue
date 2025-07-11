<template>
  <div class="m-fields-event-select">
    <m-form-table
      v-if="isOldVersion"
      name="events"
      :size="size"
      :disabled="disabled"
      :model="model"
      :config="tableConfig"
      @change="onChangeHandler"
    ></m-form-table>

    <div v-else class="fullWidth">
      <TMagicButton class="create-button" type="primary" :size="size" :disabled="disabled" @click="addEvent()"
        >添加事件</TMagicButton
      >
      <MPanel
        v-for="(cardItem, index) in model[name]"
        :key="index"
        :disabled="disabled"
        :size="size"
        :prop="`${prop}.${index}`"
        :config="actionsConfig"
        :model="cardItem"
        :label-width="config.labelWidth || '100px'"
        @change="onChangeHandler"
      >
        <template #header>
          <MFormContainer
            class="fullWidth"
            :config="eventNameConfig"
            :model="cardItem"
            :disabled="disabled"
            :size="size"
            :prop="`${prop}.${index}`"
            @change="eventNameChangeHandler"
          ></MFormContainer>
          <TMagicButton
            style="color: #f56c6c"
            link
            :icon="Delete"
            :disabled="disabled"
            :size="size"
            @click="removeEvent(index)"
          ></TMagicButton>
        </template>
      </MPanel>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { computed } from 'vue';
import { Delete } from '@element-plus/icons-vue';
import { has } from 'lodash-es';

import type { EventOption, MComponent, MContainer } from '@tmagic/core';
import { ActionType } from '@tmagic/core';
import { TMagicButton } from '@tmagic/design';
import type {
  CascaderOption,
  ChildConfig,
  CodeSelectColConfig,
  ContainerChangeEventData,
  DataSourceMethodSelectConfig,
  EventSelectConfig,
  FieldProps,
  FormState,
  OnChangeHandlerData,
  PanelConfig,
} from '@tmagic/form';
import { MContainer as MFormContainer, MPanel } from '@tmagic/form';
import { DATA_SOURCE_FIELDS_CHANGE_EVENT_PREFIX, traverseNode } from '@tmagic/utils';

import { useServices } from '@editor/hooks/use-services';
import { getCascaderOptionsFromFields } from '@editor/utils';

defineOptions({
  name: 'MFieldsEventSelect',
});

const props = defineProps<FieldProps<EventSelectConfig>>();

const emit = defineEmits<{
  change: [v: any, eventData?: ContainerChangeEventData];
}>();

const { editorService, dataSourceService, eventsService, codeBlockService, propsService } = useServices();

// 事件名称下拉框表单配置
const eventNameConfig = computed(() => {
  const defaultEventNameConfig: ChildConfig = {
    name: 'name',
    text: '事件',
    type: (mForm, { formValue }: any) => {
      if (
        props.config.src !== 'component' ||
        (formValue.type === 'page-fragment-container' && formValue.pageFragmentId)
      ) {
        return 'cascader';
      }
      return 'select';
    },
    labelWidth: '40px',
    checkStrictly: () => props.config.src !== 'component',
    valueSeparator: '.',
    options: (mForm: FormState, { formValue }: any) => {
      let events: EventOption[] | CascaderOption[] = [];

      if (props.config.src === 'component') {
        events = eventsService.getEvent(formValue.type);

        if (formValue.type === 'page-fragment-container' && formValue.pageFragmentId) {
          const pageFragment = editorService.get('root')?.items?.find((page) => page.id === formValue.pageFragmentId);
          if (pageFragment) {
            events = [
              {
                label: pageFragment.name || '页面片容器',
                value: pageFragment.id,
                children: events,
              },
            ];
            pageFragment.items.forEach((node) => {
              traverseNode<MComponent | MContainer>(node, (node) => {
                const nodeEvents = (node.type && eventsService.getEvent(node.type)) || [];

                events.push({
                  label: `${node.name}_${node.id}`,
                  value: `${node.id}`,
                  children: nodeEvents,
                });
              });
            });

            return events;
          }
        }

        return events.map((option) => ({
          text: option.label,
          value: option.value,
        }));
      }

      if (props.config.src === 'datasource') {
        // 从数据源类型中获取到相关事件
        events = dataSourceService.getFormEvent(formValue.type);
        // 从数据源类型和实例中分别获取数据以追加数据变化的事件
        const dataSource = dataSourceService.getDataSourceById(formValue.id);
        const fields = dataSource?.fields || [];
        if (fields.length > 0) {
          return [
            ...events,
            {
              label: '数据变化',
              value: DATA_SOURCE_FIELDS_CHANGE_EVENT_PREFIX,
              children: getCascaderOptionsFromFields(fields),
            },
          ];
        }

        return events;
      }
    },
  };
  return { ...defaultEventNameConfig, ...props.config.eventNameConfig };
});

// 联动类型
const actionTypeConfig = computed(() => {
  const defaultActionTypeConfig = {
    name: 'actionType',
    text: '联动类型',
    type: 'select',
    defaultValue: ActionType.COMP,
    options: () => {
      const o: {
        text: string;
        label: string;
        value: string;
        disabled?: boolean;
      }[] = [
        {
          text: '组件',
          label: '组件',
          value: ActionType.COMP,
        },
      ];

      if (!propsService.getDisabledCodeBlock()) {
        o.push({
          text: '代码',
          label: '代码',
          disabled: !Object.keys(codeBlockService.getCodeDsl() || {}).length,
          value: ActionType.CODE,
        });
      }

      if (!propsService.getDisabledDataSource()) {
        o.push({
          text: '数据源',
          label: '数据源',
          value: ActionType.DATA_SOURCE,
        });
      }

      return o;
    },
  };
  return { ...defaultActionTypeConfig, ...props.config.actionTypeConfig };
});

// 联动组件配置
const targetCompConfig = computed(() => {
  const defaultTargetCompConfig = {
    name: 'to',
    text: '联动组件',
    type: 'ui-select',
    display: (mForm: FormState, { model }: { model: Record<any, any> }) => model.actionType === ActionType.COMP,
    onChange: (MForm: FormState, v: string, { setModel }: OnChangeHandlerData) => {
      setModel('method', '');
      return v;
    },
  };
  return { ...defaultTargetCompConfig, ...props.config.targetCompConfig };
});

// 联动组件动作配置
const compActionConfig = computed(() => {
  const defaultCompActionConfig: ChildConfig = {
    name: 'method',
    text: '动作',
    type: (mForm, { model }: any) => {
      const to = editorService.getNodeById(model.to);

      if (to && to.type === 'page-fragment-container' && to.pageFragmentId) {
        return 'cascader';
      }

      return 'select';
    },
    checkStrictly: () => props.config.src !== 'component',
    display: (mForm, { model }: any) => model.actionType === ActionType.COMP,
    options: (mForm: FormState, { model }: any) => {
      const node = editorService.getNodeById(model.to);
      if (!node?.type) return [];

      let methods: EventOption[] | CascaderOption[] = [];

      methods = eventsService.getMethod(node.type, model.to);

      if (node.type === 'page-fragment-container' && node.pageFragmentId) {
        const pageFragment = editorService.get('root')?.items?.find((page) => page.id === node.pageFragmentId);
        if (pageFragment) {
          methods = [];
          pageFragment.items.forEach((node: MComponent | MContainer) => {
            traverseNode<MComponent | MContainer>(node, (node) => {
              const nodeMethods = (node.type && eventsService.getMethod(node.type, node.id)) || [];

              if (nodeMethods.length) {
                methods.push({
                  label: `${node.name}_${node.id}`,
                  value: `${node.id}`,
                  children: nodeMethods,
                });
              }
            });
          });

          return methods;
        }
      }

      return methods.map((method) => ({
        text: method.label,
        value: method.value,
      }));
    },
  };
  return { ...defaultCompActionConfig, ...props.config.compActionConfig };
});

// 代码联动配置
const codeActionConfig = computed(() => {
  const defaultCodeActionConfig: CodeSelectColConfig = {
    type: 'code-select-col',
    text: '代码块',
    name: 'codeId',
    notEditable: () => !codeBlockService.getEditStatus(),
    display: (mForm, { model }) => model.actionType === ActionType.CODE,
  };
  return { ...defaultCodeActionConfig, ...props.config.codeActionConfig };
});

// 数据源联动配置
const dataSourceActionConfig = computed(() => {
  const defaultDataSourceActionConfig: DataSourceMethodSelectConfig = {
    type: 'data-source-method-select',
    text: '数据源方法',
    name: 'dataSourceMethod',
    notEditable: () => !dataSourceService.get('editable'),
    display: (mForm, { model }) => model.actionType === ActionType.DATA_SOURCE,
  };
  return { ...defaultDataSourceActionConfig, ...props.config.dataSourceActionConfig };
});

// 兼容旧的数据格式
const tableConfig = computed(() => ({
  type: 'table',
  name: 'events',
  items: [
    {
      name: 'name',
      label: '事件名',
      type: eventNameConfig.value.type,
      options: (mForm: FormState, { formValue }: any) =>
        eventsService.getEvent(formValue.type).map((option: any) => ({
          text: option.label,
          value: option.value,
        })),
    },
    {
      name: 'to',
      label: '联动组件',
      type: 'ui-select',
    },
    {
      name: 'method',
      label: '动作',
      type: compActionConfig.value.type,
      options: (mForm: FormState, { model }: any) => {
        const node = editorService.getNodeById(model.to);
        if (!node?.type) return [];

        return eventsService.getMethod(node.type, model.to).map((option: any) => ({
          text: option.label,
          value: option.value,
        }));
      },
    },
  ],
}));

// 组件动作组表单配置
const actionsConfig = computed<PanelConfig>(() => ({
  type: 'panel',
  items: [
    {
      type: 'group-list',
      name: 'actions',
      expandAll: true,
      enableToggleMode: false,
      titlePrefix: '动作',
      items: [
        actionTypeConfig.value,
        targetCompConfig.value,
        compActionConfig.value,
        codeActionConfig.value,
        dataSourceActionConfig.value,
      ],
    },
  ],
}));

// 是否为旧的数据格式
const isOldVersion = computed(() => {
  if (props.model[props.name].length === 0) return false;
  return !has(props.model[props.name][0], 'actions');
});

// 添加事件
const addEvent = () => {
  const defaultEvent = {
    name: '',
    actions: [],
  };

  if (!props.model[props.name]) {
    props.model[props.name] = [];
  }

  emit('change', defaultEvent, {
    modifyKey: props.model[props.name].length,
  });
};

// 删除事件
const removeEvent = (index: number) => {
  if (!props.name) return;
  props.model[props.name].splice(index, 1);
  emit('change', props.model[props.name]);
};

const eventNameChangeHandler = (v: any, eventData: ContainerChangeEventData) => {
  emit('change', props.model[props.name], eventData);
};

const onChangeHandler = (v: any, eventData: ContainerChangeEventData) =>
  emit('change', props.model[props.name], eventData);
</script>
