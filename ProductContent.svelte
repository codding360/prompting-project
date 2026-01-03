<script lang="ts">
  import Icon from "@iconify/svelte";
  import { toast } from "svelte-sonner";
  import { onMount } from "svelte";

  // Components
  import PageHeader from "$lib/components/PageHeader.svelte";
  import ProductList from "$lib/features/products/ProductList.svelte";
  import ProductDialogs from "./ProductDialogs.svelte";
  import { confirmAction } from "$lib/utils/confirm.action";

  // Stores
  import {
    productCreateDialogStore,
    productSelectionStore,
  } from "$lib/store/dialogs.store";

  // Types
  import type { ProductType } from "$lib/types/product";
  import { ProductStatus } from "$lib/types/product";

  // States
  let isUpdating = $state(false);
  let selectionState = $derived($productSelectionStore);
  let refreshTrigger = $state(0);
  let loading = $state(false);
  let products = $state<ProductType[]>([]);

  // API
  import { productApi } from "$lib/api/product";
  import { storeState } from "$lib/store/store.store";

  // Functions
  function toggleSelection() {
    if (selectionState.isSelecting) {
      productSelectionStore.stopSelecting();
    } else {
      productSelectionStore.startSelecting();
    }
  }

  async function bulkUpdateProductsStatus(status: ProductStatus) {
    if (selectionState.selectedIds.size === 0 || !status) return;
    const confirmMessage = `Вы уверены, что хотите ${status === ProductStatus.ACTIVE ? "активировать" : "деактивировать"} ${selectionState.selectedIds.size} товар(ов)?`;
    await confirmAction({
      message: confirmMessage,
      onConfirm: async () => {
        const data = await productApi.bulkUpdateProductsStatus(Array.from(selectionState.selectedIds), status);
        if (!data) {
          toast.error("Ошибка при обновлении статуса товаров");
        } else {
          toast.success(`Успешно обновлен статус: ${data.length ?? 0} товар(ов)`);
          productSelectionStore.stopSelecting();
          refreshTrigger++;
        }
      },
      onCancel: async () => Promise.resolve(),
    });
  }

  async function fetchProducts() {
    try {
      loading = true;
      const data = await productApi.getProducts();
      products = data ?? [];
    } catch (error) {
      toast.error("Ошибка загрузки продуктов");
    } finally {
      loading = false;
    }
  }

  // Lifecycle
  onMount(() => {
    $storeState?.id && fetchProducts();
  });

  $effect(() => {
    if (refreshTrigger > 0) {
      fetchProducts();
    }
  });
</script>

<div class="flex flex-col h-full overflow-hidden">
  {#if products.length > 0}
    <PageHeader>
      {#snippet leftActions()}
        {#if selectionState.isSelecting}
          <div class="flex items-center gap-2">
            <button
              type="button"
              class="btn btn-outline btn-primary btn-md"
              onclick={() => productSelectionStore.stopSelecting()}
            >
              <span class="block sm:hidden">
                <Icon icon="mdi:close" class="w-5 h-5" />
              </span>
              <span class="hidden sm:block">Отмена</span>
            </button>

            <span class="text-sm text-base-content/60 hidden sm:block">
              Выбрано: {selectionState.selectedIds.size}
            </span>
          </div>
        {:else}
          <button
            type="button"
            class="btn btn-outline btn-primary btn-md"
            onclick={toggleSelection}
          >
            <span class="block sm:hidden">
              <Icon icon="mdi:check-all" class="w-5 h-5" />
            </span>
            <span class="hidden sm:block">Выбрать</span>
          </button>
        {/if}
      {/snippet}

      {#snippet rightActions()}
        {#if !selectionState.isSelecting}
          <button
            type="button"
            class="btn btn-primary btn-md"
            onclick={() => productCreateDialogStore.open()}
          >
            <Icon icon="mdi:plus" class="w-5 h-5" />
            <span class="hidden sm:inline">Создать</span>
          </button>
        {/if}

        {#if selectionState.isSelecting}
          <button
            type="button"
            class="btn btn-error btn-md"
            disabled={isUpdating || selectionState.selectedIds.size === 0}
            onclick={() => bulkUpdateProductsStatus(ProductStatus.DEACTIVATED)}
          >
            <span class="block sm:hidden">
              <Icon icon="mdi:eye-off" class="w-5 h-5" />
            </span>
            <span class="hidden sm:block">
              Деактивировать ({selectionState.selectedIds.size})
            </span>
          </button>

          <button
            type="button"
            class="btn btn-success btn-md"
            disabled={isUpdating || selectionState.selectedIds.size === 0}
              onclick={() => bulkUpdateProductsStatus(ProductStatus.ACTIVE)}
          >
            <span class="block sm:hidden">
              <Icon icon="mdi:eye" class="w-5 h-5" />
            </span>
            <span class="hidden sm:block">
              Активировать ({selectionState.selectedIds.size})
            </span>
          </button>
        {/if}
      {/snippet}
    </PageHeader>
  {/if}

  <ProductList {products} {loading} {refreshTrigger} />
  <ProductDialogs onProductChange={() => refreshTrigger++} />
</div>
