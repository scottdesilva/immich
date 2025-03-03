<script lang="ts">
  import { goto } from '$app/navigation';
  import empty3Url from '$lib/assets/empty-3.svg';
  import LinkButton from '$lib/components/elements/buttons/link-button.svelte';
  import Icon from '$lib/components/elements/icon.svelte';
  import UserPageLayout from '$lib/components/layouts/user-page-layout.svelte';
  import DeleteAssets from '$lib/components/photos-page/actions/delete-assets.svelte';
  import RestoreAssets from '$lib/components/photos-page/actions/restore-assets.svelte';
  import SelectAllAssets from '$lib/components/photos-page/actions/select-all-assets.svelte';
  import AssetGrid from '$lib/components/photos-page/asset-grid.svelte';
  import AssetSelectControlBar from '$lib/components/photos-page/asset-select-control-bar.svelte';
  import EmptyPlaceholder from '$lib/components/shared-components/empty-placeholder.svelte';
  import {
    NotificationType,
    notificationController,
  } from '$lib/components/shared-components/notification/notification';
  import { AppRoute } from '$lib/constants';
  import { AssetStore } from '$lib/stores/assets.store';
  import { featureFlags, serverConfig } from '$lib/stores/server-config.store';
  import { handleError } from '$lib/utils/handle-error';
  import { emptyTrash, restoreTrash } from '@immich/sdk';
  import { mdiDeleteForeverOutline, mdiHistory } from '@mdi/js';
  import type { PageData } from './$types';
  import { handlePromiseError } from '$lib/utils';
  import { dialogController } from '$lib/components/shared-components/dialog/dialog';
  import { t } from 'svelte-i18n';
  import { onDestroy } from 'svelte';
  import { AssetInteraction } from '$lib/stores/asset-interaction.svelte';

  interface Props {
    data: PageData;
  }

  let { data }: Props = $props();

  if (!$featureFlags.trash) {
    handlePromiseError(goto(AppRoute.PHOTOS));
  }

  const options = { isTrashed: true };
  const assetStore = new AssetStore(options);
  const assetInteraction = new AssetInteraction();

  const handleEmptyTrash = async () => {
    const isConfirmed = await dialogController.show({
      prompt: $t('empty_trash_confirmation'),
    });

    if (!isConfirmed) {
      return;
    }

    try {
      const { count } = await emptyTrash();

      notificationController.show({
        message: $t('assets_permanently_deleted_count', { values: { count } }),
        type: NotificationType.Info,
      });

      // reset asset grid (TODO fix in asset store that it should reset when it is empty)
      await assetStore.updateOptions(options);
    } catch (error) {
      handleError(error, $t('errors.unable_to_empty_trash'));
    }
  };

  const handleRestoreTrash = async () => {
    const isConfirmed = await dialogController.show({
      prompt: $t('assets_restore_confirmation'),
    });

    if (!isConfirmed) {
      return;
    }
    try {
      const { count } = await restoreTrash();
      notificationController.show({
        message: $t('assets_restored_count', { values: { count } }),
        type: NotificationType.Info,
      });

      // reset asset grid (TODO fix in asset store that it should reset when it is empty)
      await assetStore.updateOptions(options);
    } catch (error) {
      handleError(error, $t('errors.unable_to_restore_trash'));
    }
  };

  onDestroy(() => {
    assetStore.destroy();
  });
</script>

{#if assetInteraction.selectionActive}
  <AssetSelectControlBar
    assets={assetInteraction.selectedAssets}
    clearSelect={() => assetInteraction.clearMultiselect()}
  >
    <SelectAllAssets {assetStore} {assetInteraction} />
    <DeleteAssets force onAssetDelete={(assetIds) => assetStore.removeAssets(assetIds)} />
    <RestoreAssets onRestore={(assetIds) => assetStore.removeAssets(assetIds)} />
  </AssetSelectControlBar>
{/if}

{#if $featureFlags.loaded && $featureFlags.trash}
  <UserPageLayout hideNavbar={assetInteraction.selectionActive} title={data.meta.title} scrollbar={false}>
    {#snippet buttons()}
      <div class="flex place-items-center gap-2">
        <LinkButton onclick={handleRestoreTrash} disabled={assetInteraction.selectionActive}>
          <div class="flex place-items-center gap-2 text-sm">
            <Icon path={mdiHistory} size="18" />
            {$t('restore_all')}
          </div>
        </LinkButton>
        <LinkButton onclick={() => handleEmptyTrash()} disabled={assetInteraction.selectionActive}>
          <div class="flex place-items-center gap-2 text-sm">
            <Icon path={mdiDeleteForeverOutline} size="18" />
            {$t('empty_trash')}
          </div>
        </LinkButton>
      </div>
    {/snippet}

    <AssetGrid enableRouting={true} {assetStore} {assetInteraction}>
      <p class="font-medium text-gray-500/60 dark:text-gray-300/60 p-4">
        {$t('trashed_items_will_be_permanently_deleted_after', { values: { days: $serverConfig.trashDays } })}
      </p>
      {#snippet empty()}
        <EmptyPlaceholder text={$t('trash_no_results_message')} src={empty3Url} />
      {/snippet}
    </AssetGrid>
  </UserPageLayout>
{/if}
