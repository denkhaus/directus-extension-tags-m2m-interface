<template>
	<v-notice
		v-if="!relationInfo || !relationInfo.junctionCollection.collection || !relationInfo.relatedCollection.collection"
		type="warning"
	>
		{{ t('relationship_not_setup') }}
	</v-notice>

	<v-notice v-else-if="!props.referencingField" type="warning full">
		<div>
			<p>{{ t('display_template_not_setup') }}</p>
			<ul>
				<li>
					<strong>{{ t('corresponding_field') }}</strong>
					<span>:&nbsp;</span>
					{{ t('errors.NOT_NULL_VIOLATION') }}
				</li>
			</ul>
		</div>
	</v-notice>

	<template v-else>
		<v-menu v-if="selectAllowed" v-model="menuActive" attached full-height>
			<template #activator>
				<v-input
					v-model="localInput"
					:placeholder="placeholder || t('search_items')"
					:disabled="disabled"
					@keydown="onInputKeyDown"
					@focus="menuActive = true"
				>
					<template v-if="iconLeft" #prepend>
						<v-icon v-if="iconLeft" :name="iconLeft" />
					</template>

					<template #append>
						<v-icon v-if="iconRight" :name="iconRight" />
					</template>
				</v-input>
			</template>

			<v-list v-if="!disabled && (showAddCustom || suggestedItems.length)">
				<v-list-item v-if="showAddCustom" clickable @click="stageLocalInput">
					<v-list-item-content v-tooltip="t('interfaces.tags.add_tags')" class="add-custom">
						{{
							t('field_in_collection', {
								field: localInput,
								collection: isMulti ? t('select_all') : t('create_item'),
							})
						}}
					</v-list-item-content>
				</v-list-item>

				<v-divider v-if="showAddCustom && suggestedItems.length" />

				<template v-if="suggestedItems.length">
					<v-list-item
						v-for="(item, index) in suggestedItems"
						:key="item[relationInfo.relatedPrimaryKeyField.field]"
						:active="index === suggestedItemsSelected"
						clickable
						@click="() => stageItemObject(item)"
					>
						<v-list-item-content>
							{{ item[props.referencingField] }}
						</v-list-item-content>
					</v-list-item>
				</template>
			</v-list>

			<v-list v-else-if="!disabled && localInput && !createAllowed">
				<v-list-item class="no-items">
					{{ t('no_items') }}
				</v-list-item>
			</v-list>
		</v-menu>

		<v-skeleton-loader v-if="loading" type="block-list-item" />

		<div v-else-if="items.length" class="tags">
			<v-chip
				v-for="item in items"
				:key="item[relationInfo.junctionField.field][props.referencingField]"
				v-tooltip="t('remove')"
				:disabled="disabled || !selectAllowed"
				class="tag clickable"
				small
				label
				clickable
				@click="deleteItem(item)"
			>
				{{ item[relationInfo.junctionField.field][props.referencingField] }}
			</v-chip>
		</div>
	</template>
</template>

<script lang="ts" setup>
import { computed, ref, toRefs, Ref, watch, toRaw } from 'vue';
import { useI18n } from 'vue-i18n';
import { debounce, partition, get, every } from 'lodash';
import { Filter } from '@directus/shared/types';
import { useApi, useStores } from '@directus/shared/composables';
import { parseFilter, getEndpoint } from '@directus/shared/utils';
import { useRelationM2M } from './use-relations';
import * as gql from 'gql-query-builder';

type RelationFK = string | number | BigInt;
type RelationItem = RelationFK | Record<string, any>;

const props = withDefaults(
	defineProps<{
		value?: RelationItem[];
		primaryKey: string | number;
		collection: string;
		field: string;
		placeholder?: string | null;
		disabled?: boolean;
		allowCustom?: boolean;
		allowMultiple?: boolean;
		filter?: Filter | null;
		referencingField: string;
		sortField?: string | null;
		sortDirection?: string | null;
		iconLeft?: string | null;
		iconRight?: string | null;
	}>(),
	{
		value: () => [],
		disabled: false,
		filter: () => null,
		allowCustom: true,
		sortDirection: 'desc',
		iconRight: 'local_offer',
	}
);

const emit = defineEmits(['input']);

const { t } = useI18n();
const { value, collection, field } = toRefs(props);
const { relationInfo } = useRelationM2M(collection, field, useStores());
const { usePermissionsStore, useUserStore } = useStores();
const { currentUser } = useUserStore();
const { hasPermission } = usePermissionsStore();

const relCollName = get(relationInfo, 'value.relatedCollection.collection');
const relPrimaryKeyFieldName = get(relationInfo, 'value.relatedPrimaryKeyField.field');
const junCollName = get(relationInfo, 'value.junctionCollection.collection');
const junPrimaryKeyName = get(relationInfo, 'value.junctionPrimaryKeyField.field');
const junFieldName = get(relationInfo, 'value.junctionField.field');
const adminAccess = get(currentUser, 'role.admin_access', false);

const createAllowed = computed(() => {
	if (adminAccess) return true;
	if (!every([relCollName, junCollName, props.allowCustom])) return false;
	return hasPermission(relCollName, 'create') && hasPermission(junCollName, 'create');
});

const selectAllowed = computed(() => {
	if (adminAccess) return true;
	if (!junCollName) return false;
	return hasPermission(junCollName, 'create');
});

const localInput = ref<string>('');
const menuActive = ref<boolean>(false);
const suggestedItems = ref<Record<string, any>[]>([]);
const suggestedItemsSelected = ref<number | null>(null);
const api = useApi();

const fetchFields = [relPrimaryKeyFieldName];
if (props.referencingField && props.referencingField !== relPrimaryKeyFieldName) {
	fetchFields.push(props.referencingField);
}

const { items, loading } = usePreviews(value);

const showAddCustom = computed(
	() =>
		createAllowed.value &&
		localInput.value?.trim() &&
		!itemValueAvailable(localInput.value) &&
		!itemValueStaged(localInput.value)
);

const isMulti = computed(() => props.allowMultiple && fromSeparatedTag(localInput.value));

watch(
	localInput,
	debounce((newValue: string) => {
		menuActive.value = !!newValue;
		refreshSuggestions(newValue);
	}, 300)
);

function emitter(newValue: RelationItem[] | null) {
	// console.log('newValue', newValue);
	emit('input', newValue);
}

function deleteItem(item: any) {
	// console.log('item:', toRaw(item));
	// console.log('value:', toRaw(value.value));
	// console.log('junPrimaryKeyName:', junPrimaryKeyName);

	if (!every([value.value, Array.isArray(value.value), junPrimaryKeyName])) return;

	var remItems: RelationItem[];
	if (junPrimaryKeyName! in item) {
		remItems = value.value.filter((x: RelationItem) => x !== parseInt(item[junPrimaryKeyName!]));
	} else {
		remItems = value.value.filter((x: RelationItem) => x !== item);
	}

	emitter(remItems);
}

function stageItemObject(item: Record<string, RelationItem>) {
	if (junFieldName) {
		emitter([...(props.value || []), { [junFieldName]: item }]);
	}
}

async function stageLocalInput() {
	if (!props.referencingField) return;

	const value = localInput.value?.trim();
	for (const valueTag of fromSeparatedTag(value)) {
		await stageValue(valueTag.trim());
	}

	localInput.value = '';
}

function fromSeparatedTag(input: string): string[] {
	if (!props.allowMultiple) return [input];

	return input
		.split(/[;,]/)
		.map((x) => x.trim())
		.filter((x) => !!x);
}

async function stageValue(value: string) {
	if (!value || itemValueStaged(value)) return;

	const cachedItem = suggestedItems.value.find((item) => item[props.referencingField] === value);
	if (cachedItem) {
		stageItemObject(cachedItem);
		return;
	}

	try {
		const item = await findByKeyword(value);
		if (item) {
			stageItemObject(item);
		} else if (createAllowed.value) {
			stageItemObject({ [props.referencingField]: value });
		}
	} catch (err: any) {
		window.console.warn(err);
	}
}

function itemValueStaged(value: string): boolean {
	if (!every([value, props.referencingField, junFieldName])) return false;
	return !!items.value.find((item) => item[junFieldName!][props.referencingField] === value);
}

function itemValueAvailable(value: string): boolean {
	if (!every([value || props.referencingField])) return false;
	return !!suggestedItems.value.find((item) => item[props.referencingField] === value);
}

async function refreshSuggestions(keyword: string) {
	suggestedItemsSelected.value = null;
	if (!every([keyword, junFieldName, relPrimaryKeyFieldName, relCollName]) || keyword.length === 0) {
		suggestedItems.value = [];
		return;
	}

	const currentIds = items.value
		.map((item: RelationItem): RelationFK => item[junFieldName!][relPrimaryKeyFieldName])
		.filter((id: RelationFK) => id === 0 || !!id);

	const filters = [
		props.filter && parseFilter(props.filter, null),
		currentIds.length > 0 && {
			[relPrimaryKeyFieldName!]: {
				_nin: currentIds.join(','),
			},
		},
		{
			_or: fromSeparatedTag(keyword).map((value) => {
				return {
					[props.referencingField]: {
						_contains: value,
					},
				};
			}),
		},
	].filter(Boolean);

	const query = {
		params: {
			limit: 10,
			fields: fetchFields,
			filter: {
				_and: filters,
			},
			...getSortingQuery(),
		},
	};

	const response = await api.get(getEndpoint(relCollName!), query);
	if (response?.data?.data && Array.isArray(response.data.data)) {
		suggestedItems.value = response.data.data;
	} else {
		suggestedItems.value = [];
	}
}

async function findByKeyword(keyword: string): Promise<Record<string, any> | null> {
	if (!every(relCollName, keyword)) return null;
	const response = await api.get(getEndpoint(relCollName!), {
		params: {
			limit: 1,
			fields: fetchFields,
			filter: {
				[props.referencingField]: {
					_eq: keyword,
				},
			},
		},
	});

	return response?.data?.data?.pop() || null;
}

function usePreviews(value: Ref<RelationItem[]>) {
	const items = ref<any[]>([]);
	const loading = ref<boolean>(value.value && value.value.length > 0);

	if (!relationInfo.value) return { items, loading };

	// const relationalFetchFields = [
	// 	relationInfo.value.junctionPrimaryKeyField.field,
	// 	...fetchFields.map((field) => relationInfo.value!.junctionField.field + '.' + field),
	// ];

	watch(
		value,
		debounce((value: RelationItem[]) => update(value), 300)
	);

	if (value.value && Array.isArray(value.value)) {
		update(value.value);
	}

	return { items, loading };

	async function update(value: RelationItem[]) {
		const [ids, staged] = partition(value || [], (item: RelationItem) => typeof item !== 'object');

		if (!every([junPrimaryKeyName, junFieldName, junCollName]) || !ids.length) {
			items.value = [...staged];
			return;
		}

		const cached = items.value.filter(
			(item: RelationItem) =>
				typeof item === 'object' && item[junPrimaryKeyName!] && ids.includes(item[junPrimaryKeyName!])
		);

		if (cached.length === ids.length) {
			items.value = [...cached, ...staged];
			return;
		}

		loading.value = true;

		let fields = {};

		fetchFields.forEach((field) => {
			if (!fields[junFieldName!]) fields[junFieldName!] = [];
			fields[junFieldName!].push(field);
		});

		const queryObj = {
			operation: junCollName!,
			fields: [junPrimaryKeyName!, fields],
			variables: {
				limit: ids.length,
				filter: {
					value: { id: { _in: ids } },
					type: `${junCollName!}_filter`,
				},
				sort: { value: [getSortingQuery(junFieldName!).sort], list: [true], type: 'String' },
			},
		};

		const query = gql.query(queryObj);

		const response = await api.post('/graphql', query, {
			headers: {
				'Content-Type': 'application/json',
			},
		});

		const res = get(response, `data.data.${junCollName}`, null);
		if (res && Array.isArray(res)) {
			items.value = [...res, ...staged];
		} else {
			items.value = [...staged];
		}

		loading.value = false;
	}
}

function getSortingQuery(path?: string): { sort: string } {
	const fieldName = props.sortField ? props.sortField : relationInfo.value!.relatedPrimaryKeyField.field;
	const field = path ? `${path}.${fieldName}` : fieldName;
	return {
		sort: props.sortDirection === 'desc' ? `-${field}` : field,
	};
}

/**
 * Keyboard shortcuts and keybindings
 * @param {KeyboardEvent} event
 */
async function onInputKeyDown(event: KeyboardEvent) {
	if (event.key === 'Escape' && !menuActive.value && localInput.value) {
		localInput.value = '';
		return;
	}

	if (event.key === 'Escape') {
		event.preventDefault();
		menuActive.value = false;
		return;
	}

	if (event.key === 'Enter') {
		event.preventDefault();
		if (suggestedItemsSelected.value !== null && suggestedItems.value[suggestedItemsSelected.value]) {
			stageItemObject(suggestedItems.value[suggestedItemsSelected.value]);
		} else if (createAllowed.value) {
			stageLocalInput();
		}
		return;
	}

	if (event.key === 'ArrowUp' || (event.key === 'Tab' && event.shiftKey)) {
		event.preventDefault();
		if (suggestedItems.value.length < 1) return;
		// Select previous from the list, if on top, then go last.
		suggestedItemsSelected.value =
			suggestedItemsSelected.value === null ||
			suggestedItemsSelected.value < 1 ||
			!suggestedItems.value[suggestedItemsSelected.value]
				? suggestedItems.value.length - 1
				: suggestedItemsSelected.value - 1;
		return;
	}

	if (event.key === 'Tab') {
		if (!menuActive.value) {
			localInput.value = '';
			return;
		}

		if (!localInput.value && suggestedItems.value.length < 1) return;
	}

	if (event.key === 'ArrowDown' || event.key === 'Tab') {
		event.preventDefault();
		if (suggestedItems.value.length < 1) return;
		// Select next from the list, if bottom, then go first.
		suggestedItemsSelected.value =
			suggestedItemsSelected.value === null ||
			suggestedItemsSelected.value >= suggestedItems.value.length - 1 ||
			!suggestedItems.value[suggestedItemsSelected.value]
				? 0
				: suggestedItemsSelected.value + 1;
		return;
	}
}
</script>

<style scoped>
.add-custom {
	font-style: oblique;
}

.no-items {
	color: var(--foreground-subdued);
}

.tags {
	display: flex;
	flex-wrap: wrap;
	align-items: center;
	justify-content: flex-start;
	padding: 4px 0px 0px;
}

.tag {
	margin-top: 8px;
	margin-right: 8px;
	cursor: pointer;

	--v-chip-background-color: var(--primary);
	--v-chip-color: var(--foreground-inverted);
	--v-chip-background-color-hover: var(--danger);
	--v-chip-close-color: var(--v-chip-background-color);
	--v-chip-close-color-hover: var(--white);
	transition: all var(--fast) var(--transition);
}

.tag:hover {
	--v-chip-close-color: var(--white);
}
</style>
