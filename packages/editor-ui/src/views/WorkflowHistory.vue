<script setup lang="ts">
import { onBeforeMount, ref, watchEffect, computed } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import type { IWorkflowDb } from '@/Interface';
import { VIEWS, WORKFLOW_HISTORY_VERSION_RESTORE } from '@/constants';
import { useI18n, useToast } from '@/composables';
import type {
	WorkflowHistoryActionTypes,
	WorkflowVersionId,
	WorkflowHistoryRequestParams,
	WorkflowHistory,
	WorkflowVersion,
} from '@/types/workflowHistory';
import WorkflowHistoryList from '@/components/WorkflowHistory/WorkflowHistoryList.vue';
import WorkflowHistoryContent from '@/components/WorkflowHistory/WorkflowHistoryContent.vue';
import { useWorkflowHistoryStore } from '@/stores/workflowHistory.store';
import { useUIStore } from '@/stores/ui.store';
import { useWorkflowsStore } from '@/stores/workflows.store';

type WorkflowHistoryActionRecord = {
	[K in Uppercase<WorkflowHistoryActionTypes[number]>]: Lowercase<K>;
};

const enum WorkflowHistoryVersionRestoreModalActions {
	restore = 'restore',
	deactivateAndRestore = 'deactivateAndRestore',
	cancel = 'cancel',
}

const workflowHistoryActionTypes: WorkflowHistoryActionTypes = [
	'restore',
	'clone',
	'open',
	'download',
];
const WORKFLOW_HISTORY_ACTIONS = workflowHistoryActionTypes.reduce(
	(record, key) => ({ ...record, [key.toUpperCase()]: key }),
	{} as WorkflowHistoryActionRecord,
);

const route = useRoute();
const router = useRouter();
const i18n = useI18n();
const toast = useToast();
const workflowHistoryStore = useWorkflowHistoryStore();
const uiStore = useUIStore();
const workflowsStore = useWorkflowsStore();

const isListLoading = ref(true);
const requestNumberOfItems = ref(20);
const lastReceivedItemsLength = ref(0);
const editorRoute = computed(() => ({
	name: VIEWS.WORKFLOW,
	params: {
		name: route.params.workflowId,
	},
}));
const activeWorkflow = ref<IWorkflowDb | null>(null);
const workflowHistory = ref<WorkflowHistory[]>([]);
const activeWorkflowVersion = ref<WorkflowVersion | null>(null);
const activeWorkflowVersionPreview = computed<IWorkflowDb | null>(() => {
	if (activeWorkflowVersion.value && activeWorkflow.value) {
		return {
			...activeWorkflow.value,
			nodes: activeWorkflowVersion.value.nodes,
			connections: activeWorkflowVersion.value.connections,
		};
	}
	return null;
});

const loadMore = async (queryParams: WorkflowHistoryRequestParams) => {
	const history = await workflowHistoryStore.getWorkflowHistory(
		route.params.workflowId,
		queryParams,
	);
	lastReceivedItemsLength.value = history.length;
	workflowHistory.value.push(...history);
};

onBeforeMount(async () => {
	const [workflow] = await Promise.all([
		workflowsStore.fetchWorkflow(route.params.workflowId),
		loadMore({ take: requestNumberOfItems.value }),
	]);
	activeWorkflow.value = workflow;
	isListLoading.value = false;

	if (!route.params.versionId && workflowHistory.value.length) {
		await router.replace({
			name: VIEWS.WORKFLOW_HISTORY,
			params: {
				workflowId: route.params.workflowId,
				versionId: workflowHistory.value[0].versionId,
			},
		});
	}
});

const openInNewTab = (id: WorkflowVersionId) => {
	const { href } = router.resolve({
		name: VIEWS.WORKFLOW_HISTORY,
		params: {
			workflowId: route.params.workflowId,
			versionId: id,
		},
	});
	window.open(href, '_blank');
};

const openRestorationModal = async (
	isWorkflowActivated: boolean,
	formattedCreatedAt: string,
): Promise<WorkflowHistoryVersionRestoreModalActions> => {
	return new Promise((resolve, reject) => {
		const buttons = [
			{
				text: i18n.baseText('workflowHistory.action.restore.modal.button.cancel'),
				type: 'tertiary',
				action: () => {
					resolve(WorkflowHistoryVersionRestoreModalActions.cancel);
				},
			},
		];

		if (isWorkflowActivated) {
			buttons.push({
				text: i18n.baseText('workflowHistory.action.restore.modal.button.deactivateAndRestore'),
				type: 'tertiary',
				action: () => {
					resolve(WorkflowHistoryVersionRestoreModalActions.deactivateAndRestore);
				},
			});
		}

		buttons.push({
			text: i18n.baseText('workflowHistory.action.restore.modal.button.restore'),
			type: 'primary',
			action: () => {
				resolve(WorkflowHistoryVersionRestoreModalActions.restore);
			},
		});

		try {
			uiStore.openModalWithData({
				name: WORKFLOW_HISTORY_VERSION_RESTORE,
				data: {
					beforeClose: () => {
						resolve(WorkflowHistoryVersionRestoreModalActions.cancel);
					},
					isWorkflowActivated,
					formattedCreatedAt,
					buttons,
				},
			});
		} catch (error) {
			reject(error);
		}
	});
};

const onAction = async ({
	action,
	id,
	data,
}: {
	action: WorkflowHistoryActionTypes[number];
	id: WorkflowVersionId;
	data: { formattedCreatedAt: string };
}) => {
	try {
		switch (action) {
			case WORKFLOW_HISTORY_ACTIONS.OPEN:
				openInNewTab(id);
				break;
			case WORKFLOW_HISTORY_ACTIONS.DOWNLOAD:
				await workflowHistoryStore.downloadVersion(route.params.workflowId, id);
				break;
			case WORKFLOW_HISTORY_ACTIONS.CLONE:
				await workflowHistoryStore.cloneIntoNewWorkflow(route.params.workflowId, id, data);
				toast.showMessage({
					title: i18n.baseText('workflowHistory.action.clone.success.title'),
					type: 'success',
				});
				break;
			case WORKFLOW_HISTORY_ACTIONS.RESTORE:
				const workflow = await workflowsStore.fetchWorkflow(route.params.workflowId);
				const modalAction = await openRestorationModal(workflow.active, data.formattedCreatedAt);
				if (modalAction === WorkflowHistoryVersionRestoreModalActions.cancel) {
					break;
				}
				await workflowHistoryStore.restoreWorkflow(
					route.params.workflowId,
					id,
					modalAction === WorkflowHistoryVersionRestoreModalActions.deactivateAndRestore,
				);
				toast.showMessage({
					title: i18n.baseText('workflowHistory.action.restore.success.title'),
					type: 'success',
				});
				break;
		}
	} catch (error) {
		toast.showError(
			error,
			i18n.baseText('workflowHistory.action.error.title', {
				interpolate: {
					action: i18n.baseText(`workflowHistory.item.actions.${action}`).toLowerCase(),
				},
			}),
		);
	}
};

const onPreview = async ({ event, id }: { event: MouseEvent; id: WorkflowVersionId }) => {
	if (event.metaKey || event.ctrlKey) {
		openInNewTab(id);
	} else {
		await router.push({
			name: VIEWS.WORKFLOW_HISTORY,
			params: {
				workflowId: route.params.workflowId,
				versionId: id,
			},
		});
	}
};

const onUpgrade = () => {
	void uiStore.goToUpgrade('workflow-history', 'upgrade-workflow-history');
};

watchEffect(async () => {
	if (route.params.versionId) {
		const [workflow, workflowVersion] = await Promise.all([
			workflowsStore.fetchWorkflow(route.params.workflowId),
			workflowHistoryStore.getWorkflowVersion(route.params.workflowId, route.params.versionId),
		]);
		activeWorkflow.value = workflow;
		activeWorkflowVersion.value = workflowVersion;
	}
});
</script>
<template>
	<div :class="$style.view">
		<n8n-heading :class="$style.header" tag="h2" size="medium" bold>
			{{ activeWorkflow?.name }}
		</n8n-heading>
		<div :class="$style.corner">
			<n8n-heading tag="h2" size="medium" bold>
				{{ i18n.baseText('workflowHistory.title') }}
			</n8n-heading>
			<router-link :to="editorRoute">
				<n8n-button type="tertiary" icon="times" size="small" text square />
			</router-link>
		</div>
		<div :class="$style.contentComponentWrapper">
			<workflow-history-content :workflow-version="activeWorkflowVersionPreview" />
		</div>
		<div :class="$style.listComponentWrapper">
			<workflow-history-list
				:items="workflowHistory"
				:lastReceivedItemsLength="lastReceivedItemsLength"
				:activeItem="activeWorkflowVersion"
				:actionTypes="workflowHistoryActionTypes"
				:requestNumberOfItems="requestNumberOfItems"
				:shouldUpgrade="workflowHistoryStore.shouldUpgrade"
				:evaluatedPruneTime="workflowHistoryStore.evaluatedPruneTime"
				:isListLoading="isListLoading"
				@action="onAction"
				@preview="onPreview"
				@load-more="loadMore"
				@upgrade="onUpgrade"
			/>
		</div>
	</div>
</template>
<style module lang="scss">
.view {
	position: relative;
	display: grid;
	width: 100%;
	grid-template-areas: 'header corner' 'content list';
	grid-template-columns: auto 330px;
	grid-template-rows: 65px auto;
	background-color: var(--color-background-xlight);
}

.header {
	grid-area: header;
	display: flex;
	align-items: center;
	padding: 0 var(--spacing-l);
	border-bottom: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
}

.corner {
	grid-area: corner;
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 0 var(--spacing-3xs) 0 var(--spacing-s);
	background-color: var(--color-background-lighter);
	border-bottom: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
	border-left: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
}

.contentComponentWrapper {
	grid-area: content;
	position: relative;
	z-index: 1;
}

.listComponentWrapper {
	grid-area: list;
	position: relative;
	z-index: 2;

	&::before {
		content: '';
		display: block;
		position: absolute;
		top: 0;
		bottom: 0;
		left: 0;
		width: var(--border-width-base);
		background-color: var(--color-foreground-base);
	}
}
</style>
