<script lang="ts" setup>
import type { IUpdateInformation } from '@/Interface';

import type { INodeParameters, INodeProperties, NodeParameterValueType } from 'n8n-workflow';
import { deepCopy, isINodePropertyCollectionList } from 'n8n-workflow';

import { get } from 'lodash-es';

import { computed, ref, watch, onBeforeMount } from 'vue';
import { useI18n } from '@/composables/useI18n';
import {
	N8nIconButton,
	N8nSelect,
	N8nOption,
	N8nInputLabel,
	N8nText,
	N8nButton,
} from 'n8n-design-system';
import ParameterInputList from './ParameterInputList.vue';

const locale = useI18n();

export type Props = {
	nodeValues: INodeParameters;
	parameter: INodeProperties;
	path: string;
	values?: Record<string, INodeParameters[]>;
	isReadOnly?: boolean;
};

type ValueChangedEvent = {
	name: string;
	value: NodeParameterValueType;
	type?: 'optionsOrderChanged';
};

const props = withDefaults(defineProps<Props>(), {
	values: () => ({}),
	isReadOnly: false,
});

const emit = defineEmits<{
	valueChanged: [value: ValueChangedEvent];
}>();

const getPlaceholderText = computed(() => {
	const placeholder = locale.nodeText().placeholder(props.parameter, props.path);
	return placeholder ? placeholder : locale.baseText('fixedCollectionParameter.choose');
});
const mutableValues = ref({} as Record<string, INodeParameters[]>);
const selectedOption = ref<string | null | undefined>(null);
const propertyNames = computed(() => {
	return new Set(Object.keys(mutableValues.value || {}));
});
const getProperties = computed(() => {
	const returnProperties = [];
	let tempProperties;
	for (const name of propertyNames.value) {
		tempProperties = getOptionProperties(name);
		if (tempProperties !== undefined) {
			returnProperties.push(tempProperties);
		}
	}
	return returnProperties;
});
const multipleValues = computed(() => {
	return !!props.parameter.typeOptions?.multipleValues;
});
const parameterOptions = computed(() => {
	if (!isINodePropertyCollectionList(props.parameter.options)) return [];

	if (multipleValues.value) {
		return props.parameter.options;
	}

	return (props.parameter.options ?? []).filter((option) => {
		return !propertyNames.value.has(option.name);
	});
});

const sortable = computed(() => {
	return !!props.parameter.typeOptions?.sortable;
});

watch(
	() => props.values,
	(newValues: Record<string, INodeParameters[]>) => {
		mutableValues.value = deepCopy(newValues);
	},
	{ deep: true },
);

onBeforeMount(() => {
	mutableValues.value = deepCopy(props.values);
});

const deleteOption = (optionName: string, index?: number) => {
	const currentOptionsOfSameType = mutableValues.value[optionName];
	if (!currentOptionsOfSameType || currentOptionsOfSameType.length > 1) {
		// it's not the only option of this type, so just remove it.
		emit('valueChanged', {
			name: getPropertyPath(optionName, index),
			value: undefined,
		});
	} else {
		// it's the only option, so remove the whole type
		emit('valueChanged', {
			name: getPropertyPath(optionName),
			value: undefined,
		});
	}
};

const getPropertyPath = (name: string, index?: number) => {
	return `${props.path}.${name}` + (index !== undefined ? `[${index}]` : '');
};

const getOptionProperties = (optionName: string) => {
	if (isINodePropertyCollectionList(props.parameter.options)) {
		for (const option of props.parameter.options) {
			if (option.name === optionName) {
				return option;
			}
		}
	}
	return undefined;
};

const moveOptionDown = (optionName: string, index: number) => {
	if (Array.isArray(mutableValues.value[optionName])) {
		mutableValues.value[optionName].splice(
			index + 1,
			0,
			mutableValues.value[optionName].splice(index, 1)[0],
		);
	}

	const parameterData: ValueChangedEvent = {
		name: getPropertyPath(optionName),
		value: mutableValues.value[optionName],
		type: 'optionsOrderChanged',
	};

	emit('valueChanged', parameterData);
};

const moveOptionUp = (optionName: string, index: number) => {
	if (Array.isArray(mutableValues.value[optionName])) {
		mutableValues.value?.[optionName].splice(
			index - 1,
			0,
			mutableValues.value[optionName].splice(index, 1)[0],
		);
	}

	const parameterData: ValueChangedEvent = {
		name: getPropertyPath(optionName),
		value: mutableValues.value[optionName],
		type: 'optionsOrderChanged',
	};

	emit('valueChanged', parameterData);
};

const optionSelected = (optionName: string) => {
	const option = getOptionProperties(optionName);
	if (option === undefined) {
		return;
	}
	const name = `${props.path}.${option.name}`;

	const newParameterValue: INodeParameters = {};

	for (const optionParameter of option.values) {
		if (
			optionParameter.type === 'fixedCollection' &&
			optionParameter.typeOptions !== undefined &&
			optionParameter.typeOptions.multipleValues === true
		) {
			newParameterValue[optionParameter.name] = {};
		} else if (
			optionParameter.typeOptions !== undefined &&
			optionParameter.typeOptions.multipleValues === true
		) {
			// Multiple values are allowed so append option to array
			const multiValue = get(props.nodeValues, [props.path, optionParameter.name], []);

			if (Array.isArray(optionParameter.default)) {
				multiValue.push(...deepCopy(optionParameter.default));
			} else if (optionParameter.default !== '' && typeof optionParameter.default !== 'object') {
				multiValue.push(deepCopy(optionParameter.default));
			}

			newParameterValue[optionParameter.name] = multiValue;
		} else {
			// Add a new option
			newParameterValue[optionParameter.name] = deepCopy(optionParameter.default);
		}
	}

	let newValue: NodeParameterValueType;
	if (multipleValues.value) {
		newValue = get(props.nodeValues, name, []) as INodeParameters[];

		newValue.push(newParameterValue);
	} else {
		newValue = newParameterValue;
	}

	const parameterData = {
		name,
		value: newValue,
	};

	emit('valueChanged', parameterData);
	selectedOption.value = undefined;
};

const valueChanged = (parameterData: IUpdateInformation) => {
	emit('valueChanged', parameterData);
};
</script>

<template>
	<div
		class="fixed-collection-parameter"
		:data-test-id="`fixed-collection-${props.parameter?.name}`"
		@keydown.stop
	>
		<div v-if="getProperties.length === 0" class="no-items-exist">
			<N8nText size="small">{{
				locale.baseText('fixedCollectionParameter.currentlyNoItemsExist')
			}}</N8nText>
		</div>

		<div
			v-for="property in getProperties"
			:key="property.name"
			class="fixed-collection-parameter-property"
		>
			<N8nInputLabel
				v-if="property.displayName !== '' && parameter.options && parameter.options.length !== 1"
				:label="locale.nodeText().inputLabelDisplayName(property, path)"
				:underline="true"
				size="small"
				color="text-dark"
			/>
			<div v-if="multipleValues">
				<div
					v-for="(_, index) in mutableValues[property.name]"
					:key="property.name + index"
					class="parameter-item"
				>
					<div
						:class="index ? 'border-top-dashed parameter-item-wrapper ' : 'parameter-item-wrapper'"
					>
						<div v-if="!isReadOnly" class="delete-option">
							<N8nIconButton
								type="tertiary"
								text
								size="mini"
								icon="trash"
								data-test-id="fixed-collection-delete"
								:title="locale.baseText('fixedCollectionParameter.deleteItem')"
								@click="deleteOption(property.name, index)"
							></N8nIconButton>
							<N8nIconButton
								v-if="sortable && index !== 0"
								type="tertiary"
								text
								size="mini"
								icon="angle-up"
								:title="locale.baseText('fixedCollectionParameter.moveUp')"
								@click="moveOptionUp(property.name, index)"
							></N8nIconButton>
							<N8nIconButton
								v-if="sortable && index !== mutableValues[property.name].length - 1"
								type="tertiary"
								text
								size="mini"
								icon="angle-down"
								:title="locale.baseText('fixedCollectionParameter.moveDown')"
								@click="moveOptionDown(property.name, index)"
							></N8nIconButton>
						</div>
						<Suspense>
							<ParameterInputList
								:parameters="property.values"
								:node-values="nodeValues"
								:path="getPropertyPath(property.name, index)"
								:hide-delete="true"
								:is-read-only="isReadOnly"
								@value-changed="valueChanged"
							/>
						</Suspense>
					</div>
				</div>
			</div>
			<div v-else class="parameter-item">
				<div class="parameter-item-wrapper">
					<div v-if="!isReadOnly" class="delete-option">
						<N8nIconButton
							type="tertiary"
							text
							size="mini"
							icon="trash"
							data-test-id="fixed-collection-delete"
							:title="locale.baseText('fixedCollectionParameter.deleteItem')"
							@click="deleteOption(property.name)"
						></N8nIconButton>
					</div>
					<ParameterInputList
						:parameters="property.values"
						:node-values="nodeValues"
						:path="getPropertyPath(property.name)"
						:is-read-only="isReadOnly"
						class="parameter-item"
						:hide-delete="true"
						@value-changed="valueChanged"
					/>
				</div>
			</div>
		</div>

		<div v-if="parameterOptions.length > 0 && !isReadOnly" class="controls">
			<N8nButton
				v-if="parameter.options && parameter.options.length === 1"
				type="tertiary"
				block
				data-test-id="fixed-collection-add"
				:label="getPlaceholderText"
				@click="optionSelected(parameter.options[0].name)"
			/>
			<div v-else class="add-option">
				<N8nSelect
					v-model="selectedOption"
					:placeholder="getPlaceholderText"
					size="small"
					filterable
					@update:model-value="optionSelected"
				>
					<N8nOption
						v-for="item in parameterOptions"
						:key="item.name"
						:label="locale.nodeText().collectionOptionDisplayName(parameter, item, path)"
						:value="item.name"
					></N8nOption>
				</N8nSelect>
			</div>
		</div>
	</div>
</template>

<style scoped lang="scss">
.fixed-collection-parameter {
	padding-left: var(--spacing-s);

	.delete-option {
		display: flex;
		flex-direction: column;
	}

	.controls {
		:deep(.button) {
			font-weight: var(--font-weight-normal);
			--button-font-color: var(--color-text-dark);
			--button-border-color: var(--color-foreground-base);
			--button-background-color: var(--color-background-base);

			--button-hover-font-color: var(--color-text-dark);
			--button-hover-border-color: var(--color-foreground-base);
			--button-hover-background-color: var(--color-background-base);

			--button-active-font-color: var(--color-text-dark);
			--button-active-border-color: var(--color-foreground-base);
			--button-active-background-color: var(--color-background-base);

			--button-focus-font-color: var(--color-text-dark);
			--button-focus-border-color: var(--color-foreground-base);
			--button-focus-background-color: var(--color-background-base);

			&:active,
			&.active,
			&:focus {
				outline: none;
			}
		}
	}
}

.fixed-collection-parameter-property {
	margin: var(--spacing-xs) 0;
}

.parameter-item:hover > .parameter-item-wrapper > .delete-option {
	opacity: 1;
}

.parameter-item {
	position: relative;
	padding: 0 0 0 1em;

	+ .parameter-item {
		.parameter-item-wrapper {
			.delete-option {
				top: 14px;
			}
		}
	}
}

.border-top-dashed {
	border-top: 1px dashed #999;
}

.no-items-exist {
	margin: var(--spacing-xs) 0;
}
</style>
