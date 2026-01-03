<script lang="ts">
  // Libraries
  import { toast } from "svelte-sonner";
  import { onMount } from "svelte";
  import ProductForm from "./ProductForm.svelte";

  // Components
  import Dialog from "$lib/components/Dialog.svelte";

  // Stores
  import { productCreateDialogStore } from "../../store/dialogs.store";

  // Types
  import { ProductStatus, type ProductCreateType } from "$lib/types/product";
  import { productApi } from "$lib/api/product";

  let { onProductChange }: { onProductChange?: () => void } = $props();

  let dialog: { open: () => void; close: () => void } | null = $state(null);
  let isLargeScreen = $state(false);
  let isLoading = $state(false);
  let error: string | null = $state(null);

  let data: ProductCreateType = $state({
    title: "",
    description: "",
    tags: [],
    cost: undefined,
    moq: undefined,
    bulkPrice: undefined,
    code: "",
    price: undefined,
    images: [],
  });

  onMount(() => {
    const mediaQuery = window.matchMedia("(min-width: 1024px)");
    isLargeScreen = mediaQuery.matches;
    
    const handleResize = (e: MediaQueryListEvent) => {
      isLargeScreen = e.matches;
    };
    
    mediaQuery.addEventListener("change", handleResize);
    
    // Generate product code on mount
    generateProductCode();
    
    // Open dialog after a short delay to ensure it's fully initialized
    const timer = setTimeout(() => {
      dialog?.open();
    }, 50);
    
    return () => {
      mediaQuery.removeEventListener("change", handleResize);
      clearTimeout(timer);
    };
  });

  async function generateProductCode() {
    if (data.code) return; // Don't regenerate if already has a code
    
    isLoading = true;
    try {
      const code = await productApi.generateProductCode();
      if (code) {
        data.code = code;
      }
    } catch (error) {
      toast.error("Не удалось сгенерировать код продукта");
    } finally {
      isLoading = false;
    }
  }

  function resetForm() {
    data = {
      title: "",
      code: "",
      price: undefined,
      description: "",
      tags: [],
      cost: undefined,
      moq: undefined,
      bulkPrice: undefined,
      images: [],
    } as ProductCreateType;
  }

  async function submit(status: ProductStatus) {
    if (!status) return;
    
    try {
      isLoading = true;
      error = null;
      
      const product = await productApi.createProduct(data);
      if (!product) {
        toast.error("Не удалось создать продукт");
        return;
      }
    
      resetForm();
      productCreateDialogStore.close();
      toast.success("Продукт успешно создан");
      onProductChange?.();
    } catch (error) {
      toast.error("Не удалось создать продукт", {
        description: error instanceof Error ? error.message : "Неизвестная ошибка",
      });
    } finally {
      isLoading = false;
    }
  }

  function updateImages(publicUrlImage: string) {
    data.images = [...data.images, publicUrlImage];
  }
</script>

<Dialog
  bind:this={dialog}
  title="Создание продукта"
  position="center"
  onclose={() => productCreateDialogStore.close()}
  class={isLargeScreen ? "bg-base-300" : "lg:modal-box-mobile bg-base-300"}
>
  {#snippet children()}
    <div class={isLargeScreen ? "max-h-[60vh] overflow-y-auto scrollbar-hide" : "h-[calc(100vh-180px)] overflow-y-auto scrollbar-hide"}>
      <ProductForm 
        {data} 
        {isLoading} 
        mode="create" 
        onupdateimages={updateImages}
      />
    </div>
  {/snippet}

  {#snippet actions()}
    <button class="btn btn-outline" type="button" onclick={() => submit(ProductStatus.DRAFT)} disabled={isLoading}>
      Черновик
    </button>
    <button class="btn btn-primary" type="button" onclick={() => submit(ProductStatus.ACTIVE)} disabled={isLoading}>
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