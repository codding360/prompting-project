<script lang="ts">
  import { toast } from "svelte-sonner";
  import { onMount } from "svelte";

  // Components
  import Dialog from "$lib/components/Dialog.svelte";
  import ProductForm from "./ProductForm.svelte";

  // Types
  import type { ProductEditType } from "$lib/types/product";

  // Stores
  import { productEditDialogStore } from "../../store/dialogs.store";

  let { 
    product,
    onProductChange
  }: { 
    product: ProductEditType;
    onProductChange?: () => void;
  } = $props();

  let dialog: { open: () => void; close: () => void } | null = $state(null);
  let isLargeScreen = $state(false);
  let isLoading = $state(false);
  let imageFiles: File[] = $state([]);
  let imagePreviews: string[] = $state([]);
  
  // Initialize with null values to prevent undefined bind errors
  let data: ProductEditType = $state({
    id: "",
    title: "",
    code: "",
    price: undefined,
    description: "",
    tags: [],
    cost: undefined,
    moq: undefined,
    bulkPrice: undefined,
    images: [],
  });

  // Initialize data from product
  $effect(() => {
    data.title = product.title;
    data.code = product.code;
    data.price = product.price;
    data.description = product.description;
    data.tags = product.tags;
    data.cost = product.cost;
    data.moq = product.moq;
    data.bulkPrice = product.bulkPrice;
    data.images = product.images ?? [];
    // Set existing images as previews
    imagePreviews = product.images ?? [];
    imageFiles = [];
  });

  onMount(() => {
    const mediaQuery = window.matchMedia("(min-width: 1024px)");
    isLargeScreen = mediaQuery.matches;
    
    const handleResize = (e: MediaQueryListEvent) => {
      isLargeScreen = e.matches;
    };
    
    mediaQuery.addEventListener("change", handleResize);
    
    // Open dialog after a short delay to ensure it's fully initialized
    const timer = setTimeout(() => {
      dialog?.open();
    }, 50);
    
    return () => {
      mediaQuery.removeEventListener("change", handleResize);
      clearTimeout(timer);
    };
  });

  function close() {
    productEditDialogStore.close();
  }

  async function submit(status: 'DRAFT' | 'ACTIVE' = 'DRAFT') {
    if (!product) return;
    
    try {
      isLoading = true;
      
      const formData = new FormData();
      formData.append('title', data.title);
      if (data.code) formData.append('code', data.code);
      formData.append('price', data.price?.toString() ?? "");
      formData.append('status', status);
      
      // Add new images
      imageFiles.forEach(file => {
        formData.append('images', file);
      });
      
      // Add existing image URLs
      const existingImages = imagePreviews.filter(url => url.startsWith('http'));
      if (existingImages.length > 0) {
        formData.append('existingImages', JSON.stringify(existingImages));
      }
      
      // B2C fields
      if (data.description) formData.append('description', data.description);
      if (data.tags.length > 0) formData.append('tags', JSON.stringify(data.tags));
      if (data.cost) formData.append('cost', data.cost?.toString() ?? "");
      
      // B2B fields
      if (data.moq) formData.append('moq', data.moq?.toString() ?? "");
      if (data.bulkPrice) formData.append('bulkPrice', data.bulkPrice?.toString() ?? "");

      const response = await fetch(`/api/products/${product.id}`, {
        method: 'PUT',
        body: formData,
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error || 'Ошибка при обновлении продукта');
      }
      
      close();
      toast.success("Продукт успешно обновлен");
      
      // Обновляем список продуктов без перезагрузки
      onProductChange?.();
    } catch (error) {
      toast.error("Не удалось обновить продукт", {
        description: error instanceof Error ? error.message : "Неизвестная ошибка",
      });
    } finally {
      isLoading = false;
    }
  }
</script>

<Dialog
  bind:this={dialog}
  title="Редактирование продукта"
  position="center"
  onclose={() => productEditDialogStore.close()}
  class={isLargeScreen ? "bg-base-300" : "lg:modal-box-mobile bg-base-300"}
>
  {#snippet children()}
    <div class={isLargeScreen ? "max-h-[60vh] overflow-y-auto scrollbar-hide" : "h-[calc(100vh-180px)] overflow-y-auto scrollbar-hide"}>
      <ProductForm {data} {isLoading} mode="update" bind:imageFiles bind:imagePreviews />
    </div>
  {/snippet}

  {#snippet actions()}
    <button class="btn btn-outline" type="button" onclick={() => submit('DRAFT')} disabled={isLoading}>
      Черновик
    </button>
    <button class="btn btn-primary" type="button" onclick={() => submit('ACTIVE')} disabled={isLoading}>
      Опубликовать
    </button>
  {/snippet}
</Dialog>

<style>
  .scrollbar-hide {
    scrollbar-width: none;
  }
  .scrollbar-hide::-webkit-scrollbar {
    display: none;
  }
  
  :global(.modal-box.lg\:modal-box-mobile) {
    max-width: 100vw !important;
    width: 100vw !important;
    height: 100vh !important;
    max-height: 100vh !important;
    margin: 0 !important;
    border-radius: 0 !important;
  }
  
  @media (min-width: 1024px) {
    :global(.modal-box.lg\:modal-box-mobile) {
      max-width: 800px !important;
      width: auto !important;
      height: auto !important;
      max-height: 90vh !important;
      margin: auto !important;
      border-radius: var(--rounded-box) !important;
    }
  }
</style>

