<script lang="ts">
  import ButtonContextMenu from '$lib/components/shared-components/context-menu/button-context-menu.svelte';
  import MenuOption from '$lib/components/shared-components/context-menu/menu-option.svelte';
  import { assetMultiSelectManager } from '$lib/managers/asset-multi-select-manager.svelte';
  import { rotateAssetsBy, type RotatedAssetUpdate } from '$lib/utils/asset-utils';
  import { handleError } from '$lib/utils/handle-error';
  import { modalManager, toastManager } from '@immich/ui';
  import { mdiRotateLeft, mdiRotateRight } from '@mdi/js';
  import { t } from 'svelte-i18n';

  type Props = {
    onRotated?: (updates: RotatedAssetUpdate[]) => void;
  };

  let { onRotated }: Props = $props();

  const options = $derived([
    { degrees: 90, label: $t('rotate_90_clockwise'), icon: mdiRotateRight },
    { degrees: 270, label: $t('rotate_270_clockwise'), icon: mdiRotateLeft },
  ]);

  const handleRotate = async (degrees: number, label: string, icon: string) => {
    const eligible = assetMultiSelectManager.ownedAssets.filter(
      (asset) => asset.isImage && !asset.livePhotoVideoId,
    );

    if (eligible.length === 0) {
      toastManager.warning($t('rotate_no_eligible_assets'));
      return;
    }

    const isConfirmed = await modalManager.showDialog({
      title: label,
      prompt: $t('rotate_confirmation', { values: { count: eligible.length, degrees } }),
      confirmText: $t('save'),
      confirmColor: 'primary',
      icon,
    });

    if (!isConfirmed) {
      return;
    }

    try {
      const updates = await rotateAssetsBy(eligible, degrees);
      if (updates.length > 0) {
        onRotated?.(updates);
        assetMultiSelectManager.clear();
      }
    } catch (error) {
      handleError(error, $t('errors.unable_to_rotate_assets'));
    }
  };
</script>

<ButtonContextMenu icon={mdiRotateRight} title={$t('rotate_menu_title')} align="top-right" direction="left">
  {#each options as { degrees, label, icon } (degrees)}
    <MenuOption {icon} text={label} onClick={() => handleRotate(degrees, label, icon)} />
  {/each}
</ButtonContextMenu>
