<script lang="ts">
  import { productCreateDialogStore, productEditDialogStore } from "../../store/dialogs.store";
  import ProductCreateDialog from "$lib/features/products/ProductCreateDialog.svelte";
  import ProductEditDialog from "$lib/features/products/ProductEditDialog.svelte";  
  
  let { onProductChange }: { onProductChange?: () => void } = $props();
  const createDialogState = $derived($productCreateDialogStore);
  const editDialogState = $derived($productEditDialogStore);
</script>

{#if createDialogState.isOpen}
  <ProductCreateDialog {onProductChange} />
{/if}

{#if editDialogState.isOpen && editDialogState.product}
  <ProductEditDialog product={editDialogState.product} {onProductChange} />
{/if}

