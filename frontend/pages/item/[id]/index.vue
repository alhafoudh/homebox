<script setup lang="ts">
  import { useI18n } from "vue-i18n";
  import { toast } from "@/components/ui/sonner";
  import type { AnyDetail, Detail, Details } from "~~/components/global/DetailsSection/types";
  import { filterZeroValues } from "~~/components/global/DetailsSection/types";
  import type { ItemAttachment } from "~~/lib/api/types/data-contracts";
  import MdiClose from "~icons/mdi/close";
  import MdiPackageVariant from "~icons/mdi/package-variant";
  import MdiPlus from "~icons/mdi/plus";
  import MdiMinus from "~icons/mdi/minus";
  import MdiDownload from "~icons/mdi/download";
  import MdiContentCopy from "~icons/mdi/content-copy";
  import MdiDelete from "~icons/mdi/delete";
  import { Separator } from "@/components/ui/separator";
  import {
    Breadcrumb,
    BreadcrumbItem,
    BreadcrumbLink,
    BreadcrumbList,
    BreadcrumbSeparator,
  } from "@/components/ui/breadcrumb";
  import { Button, ButtonGroup, buttonVariants } from "@/components/ui/button";
  import { Dialog, DialogContent } from "@/components/ui/dialog";
  import { useDialog } from "@/components/ui/dialog-provider";
  import { Label } from "@/components/ui/label";
  import { Switch } from "@/components/ui/switch";
  import { Card } from "@/components/ui/card";
  import { DialogID } from "~/components/ui/dialog-provider/utils";

  const { t } = useI18n();

  const { openDialog, closeDialog } = useDialog();

  definePageMeta({
    middleware: ["auth"],
  });

  const route = useRoute();
  const router = useRouter();
  const api = useUserApi();

  const itemId = computed<string>(() => route.params.id as string);
  const preferences = useViewPreferences();

  const hasNested = computed<boolean>(() => {
    return route.fullPath.split("/").at(-1) !== itemId.value;
  });

  const { data: item, refresh } = useAsyncData(itemId.value, async () => {
    const { data, error } = await api.items.get(itemId.value);
    if (error) {
      toast.error(t("items.toast.failed_load_item"));
      navigateTo("/home");
      return;
    }
    return data;
  });
  onMounted(() => {
    refresh();
  });

  const lastRoute = ref(route.fullPath);
  watchEffect(() => {
    if (lastRoute.value.endsWith("edit")) {
      refresh();
    }

    lastRoute.value = route.fullPath;
  });

  async function adjustQuantity(amount: number) {
    if (!item.value) {
      return;
    }

    const newQuantity = item.value.quantity + amount;
    if (newQuantity < 0) {
      toast.error(t("items.toast.quantity_cannot_negative"));
      return;
    }

    const resp = await api.items.patch(item.value.id, {
      id: item.value.id,
      quantity: newQuantity,
    });

    if (resp.error) {
      toast.error(t("items.toast.failed_adjust_quantity"));
      return;
    }

    item.value.quantity = newQuantity;
  }

  type FilteredAttachments = {
    attachments: ItemAttachment[];
    warranty: ItemAttachment[];
    manuals: ItemAttachment[];
    receipts: ItemAttachment[];
  };

  type Photo = {
    thumbnailSrc?: string;
    originalSrc: string;
    originalType?: string;
  };

  const photos = computed<Photo[]>(() => {
    if (!item.value) {
      return [];
    }
    return (
      item.value.attachments.reduce((acc, cur) => {
        if (cur.type === "photo") {
          const photo: Photo = {
            originalSrc: api.authURL(`/items/${item.value!.id}/attachments/${cur.id}`),
            originalType: cur.mimeType,
          };
          if (cur.thumbnail) {
            photo.thumbnailSrc = api.authURL(`/items/${item.value!.id}/attachments/${cur.thumbnail.id}`);
          } else {
            photo.thumbnailSrc = photo.originalSrc; // fallback to itself if no thumbnail
          }
          acc.push(photo);
        }
        return acc;
      }, [] as Photo[]) || []
    );
  });

  const attachments = computed<FilteredAttachments>(() => {
    if (!item.value) {
      return {
        attachments: [],
        manuals: [],
        warranty: [],
        receipts: [],
      };
    }

    return item.value.attachments.reduce(
      (acc, attachment) => {
        if (attachment.type === "photo") {
          return acc;
        }
        if (attachment.type === "warranty") {
          acc.warranty.push(attachment);
        } else if (attachment.type === "manual") {
          acc.manuals.push(attachment);
        } else if (attachment.type === "receipt") {
          acc.receipts.push(attachment);
        } else {
          acc.attachments.push(attachment);
        }
        return acc;
      },
      {
        attachments: [] as ItemAttachment[],
        warranty: [] as ItemAttachment[],
        manuals: [] as ItemAttachment[],
        receipts: [] as ItemAttachment[],
      }
    );
  });

  const assetID = computed<Details>(() => {
    if (!item.value) {
      return [];
    }

    if (item.value?.assetId === "000-000") {
      return [];
    }

    return [
      {
        name: "items.asset_id",
        text: item.value?.assetId,
      },
    ];
  });

  const itemDetails = computed<Details>(() => {
    if (!item.value) {
      return [];
    }

    const ret: Details = [
      {
        name: "items.quantity",
        text: item.value?.quantity,
        slot: "quantity",
      },
      {
        name: "items.serial_number",
        text: item.value?.serialNumber,
        copyable: true,
      },
      {
        name: "items.model_number",
        text: item.value?.modelNumber,
        copyable: true,
      },
      {
        name: "items.manufacturer",
        text: item.value?.manufacturer,
        copyable: true,
      },
      {
        name: "items.insured",
        text: item.value?.insured ? "Yes" : "No",
      },
      {
        name: "items.archived",
        text: item.value?.archived ? "Yes" : "No",
      },
      {
        name: "items.notes",
        type: "markdown",
        text: item.value?.notes,
      },
      ...assetID.value,
      ...item.value.fields.map(field => {
        /**
         * Support Special URL Syntax
         */
        const url = maybeUrl(field.textValue);
        if (url.isUrl) {
          return {
            type: "link",
            name: field.name,
            text: url.text,
            href: url.url,
          } as AnyDetail;
        }

        return {
          name: field.name,
          text: field.textValue,
        };
      }),
    ];

    if (!preferences.value.showEmpty) {
      return filterZeroValues(ret);
    }

    return ret;
  });

  const showAttachments = computed(() => {
    if (preferences.value?.showEmpty) {
      return true;
    }

    return (
      attachments.value.attachments.length > 0 ||
      attachments.value.warranty.length > 0 ||
      attachments.value.manuals.length > 0 ||
      attachments.value.receipts.length > 0
    );
  });

  const attachmentDetails = computed(() => {
    const details: Detail[] = [];

    const push = (name: string, slot: string) => {
      details.push({
        name,
        text: "",
        slot,
      });
    };

    if (attachments.value.attachments.length > 0) {
      push("items.attachments", "attachments");
    }

    if (attachments.value.warranty.length > 0) {
      push("items.warranty", "warranty");
    }

    if (attachments.value.manuals.length > 0) {
      push("items.manuals", "manuals");
    }

    if (attachments.value.receipts.length > 0) {
      push("items.receipts", "receipts");
    }

    return details;
  });

  const showWarranty = computed(() => {
    if (preferences.value.showEmpty) {
      return true;
    }
    return item.value?.lifetimeWarranty || validDate(item.value?.warrantyExpires);
  });

  const warrantyDetails = computed(() => {
    const details: Details = [
      {
        name: "items.lifetime_warranty",
        text: item.value?.lifetimeWarranty ? "Yes" : "No",
      },
    ];

    if (item.value?.lifetimeWarranty) {
      details.push({
        name: "items.warranty_expires",
        text: "N/A",
      });
    } else {
      details.push({
        name: "items.warranty_expires",
        text: item.value?.warrantyExpires || "",
        type: "date",
        date: true,
      });
    }

    details.push({
      name: "items.warranty_details",
      type: "markdown",
      text: item.value?.warrantyDetails || "",
    });

    if (!preferences.value.showEmpty) {
      return filterZeroValues(details);
    }

    return details;
  });

  const showPurchase = computed(() => {
    if (preferences.value.showEmpty) {
      return true;
    }
    return item.value?.purchaseFrom || item.value?.purchasePrice !== 0 || validDate(item.value?.purchaseTime);
  });

  const purchaseDetails = computed<Details>(() => {
    const v: Details = [
      {
        name: "items.purchased_from",
        text: item.value?.purchaseFrom || "",
      },
      {
        name: "items.purchase_price",
        text: String(item.value?.purchasePrice) || "",
        type: "currency",
      },
      {
        name: "items.purchase_date",
        text: item.value?.purchaseTime || "",
        type: "date",
        date: true,
      },
    ];

    if (!preferences.value.showEmpty) {
      return filterZeroValues(v);
    }

    return v;
  });

  const showSold = computed(() => {
    if (preferences.value.showEmpty) {
      return true;
    }
    return item.value?.soldTo || item.value?.soldPrice !== 0 || validDate(item.value?.soldTime);
  });

  const soldDetails = computed<Details>(() => {
    const v: Details = [
      {
        name: "items.sold_to",
        text: item.value?.soldTo || "",
      },
      {
        name: "items.sold_price",
        text: String(item.value?.soldPrice) || "",
        type: "currency",
      },
      {
        name: "items.sold_at",
        text: item.value?.soldTime || "",
        type: "date",
        date: true,
      },
    ];

    if (!preferences.value.showEmpty) {
      return filterZeroValues(v);
    }

    return v;
  });

  const dialoged = reactive<Photo>({
    originalSrc: "",
  });

  function openImageDialog(img: Photo) {
    dialoged.originalSrc = img.originalSrc;
    dialoged.originalType = img.originalType;
    dialoged.thumbnailSrc = img.thumbnailSrc;
    openDialog(DialogID.ItemImage);
  }

  function closeImageDialog() {
    closeDialog(DialogID.ItemImage);
  }

  const currentUrl = computed(() => {
    return window.location.href;
  });

  const currentPath = computed(() => {
    return route.path;
  });

  const tabs = computed(() => {
    return [
      {
        id: "details",
        name: "global.details",
        to: `/item/${itemId.value}`,
      },
      {
        id: "log",
        name: "global.maintenance",
        to: `/item/${itemId.value}/maintenance`,
      },
      {
        id: "edit",
        name: "global.edit",
        to: `/item/${itemId.value}/edit`,
      },
    ];
  });

  const fullpath = computedAsync(async () => {
    if (!item.value) {
      return [];
    }

    const resp = await api.items.fullpath(item.value.id);
    if (resp.error) {
      toast.error(t("items.toast.failed_load_item"));
      return [];
    }

    return resp.data;
  });

  const items = computedAsync(async () => {
    if (!item.value) {
      return [];
    }

    const resp = await api.items.getAll({
      parentIds: [item.value.id],
    });

    if (resp.error) {
      toast.error(t("items.toast.failed_load_items"));
      return [];
    }

    return resp.data.items;
  });

  async function duplicateItem() {
    if (!item.value) {
      return;
    }

    const { error, data } = await api.items.create({
      name: `${item.value.name} Copy`,
      description: item.value.description,
      quantity: item.value.quantity,
      locationId: item.value.location!.id,
      parentId: item.value.parent?.id,
      labelIds: item.value.labels.map(l => l.id),
    });

    if (error) {
      toast.error(t("items.toast.failed_duplicate_item"));
      return;
    }

    // add extra fields
    const { error: updateError } = await api.items.update(data.id, {
      ...item.value,
      id: data.id,
      labelIds: data.labels.map(l => l.id),
      locationId: data.location!.id,
      name: data.name,
      assetId: data.assetId,
    });

    if (updateError) {
      toast.error(t("items.toast.failed_duplicate_item"));
      return;
    }

    navigateTo(`/item/${data.id}`);
  }

  const confirm = useConfirm();

  async function deleteItem() {
    const confirmed = await confirm.open(t("items.delete_item_confirm"));

    if (!confirmed.data) {
      return;
    }

    const { error } = await api.items.delete(itemId.value);
    if (error) {
      toast.error(t("items.toast.failed_delete_item"));
      return;
    }
    toast.success(t("items.toast.item_deleted"));
    navigateTo("/home");
  }

  async function createSubitem() {
    // setting URL Parameter that is read and immidiately removed in the Item-CreateModal
    await router.push({
      query: {
        subItemCreate: "y",
      },
    });

    openDialog(DialogID.CreateItem);
  }
</script>

<template>
  <BaseContainer v-if="item">
    <!-- set page title -->
    <Title>{{ item.name }}</Title>

    <Dialog :dialog-id="DialogID.ItemImage">
      <DialogContent class="w-auto border-transparent bg-transparent p-0" disable-close>
        <picture>
          <source :srcset="dialoged.originalSrc" :type="dialoged.originalType" />
          <img :src="dialoged.thumbnailSrc" alt="attachement image" />
        </picture>
        <a
          :class="buttonVariants({ size: 'icon' })"
          :href="dialoged.originalSrc"
          download
          class="absolute right-11 top-1"
        >
          <MdiDownload />
        </a>
        <Button size="icon" class="absolute right-1 top-1" @click="closeImageDialog">
          <MdiClose />
        </Button>
      </DialogContent>
    </Dialog>

    <section>
      <Card class="p-3">
        <header :class="{ 'mb-2': item.description }">
          <div class="flex flex-wrap items-end gap-2">
            <div
              class="mb-auto flex size-12 items-center justify-center rounded-full bg-secondary text-secondary-foreground"
            >
              <MdiPackageVariant class="size-7" />
            </div>
            <div>
              <Breadcrumb v-if="fullpath && fullpath.length > 0">
                <BreadcrumbList>
                  <BreadcrumbItem v-for="(part, idx) in fullpath" :key="part.id">
                    <BreadcrumbLink
                      v-if="idx < fullpath.length - 1"
                      as-child
                      class="text-foreground/70 hover:underline"
                    >
                      <NuxtLink :to="`/${part.type}/${part.id}`">
                        {{ part.name }}
                      </NuxtLink>
                    </BreadcrumbLink>
                    <template v-else>
                      {{ part.name }}
                    </template>
                    <BreadcrumbSeparator v-if="idx < fullpath.length - 1" :key="`sep-${part.id}`" />
                  </BreadcrumbItem>
                </BreadcrumbList>
              </Breadcrumb>
              <h1 class="text-wrap pb-1 text-2xl">
                {{ item ? item.name : "" }}
              </h1>
              <div class="flex flex-wrap gap-2 pb-1">
                <LabelChip v-for="label in item?.labels || []" :key="label.id" :label="label" size="sm" />
              </div>
              <div class="flex flex-wrap gap-1 text-wrap text-xs">
                <div>
                  {{ $t("items.created_at") }}
                  <DateTime :date="item?.createdAt" />
                </div>
                -
                <div>
                  {{ $t("items.updated_at") }}
                  <DateTime :date="item?.updatedAt" />
                </div>
              </div>
            </div>
            <div class="ml-auto mt-2 flex flex-wrap items-center justify-between gap-3">
              <LabelMaker
                v-if="typeof item.assetId === 'string' && item.assetId != ''"
                :id="item.assetId"
                type="asset"
              />
              <LabelMaker v-else :id="item.id" type="item" />
              <Button class="w-9 md:w-auto" :aria-label="$t('global.create_subitem')" @click="createSubitem">
                <MdiPlus />
                <span class="hidden md:inline">{{ $t("global.create_subitem") }}</span>
              </Button>
              <Button class="w-9 md:w-auto" :aria-label="$t('global.duplicate')" @click="duplicateItem">
                <MdiContentCopy />
                <span class="hidden md:inline">{{ $t("global.duplicate") }}</span>
              </Button>
              <Button variant="destructive" class="w-9 md:w-auto" :aria-label="$t('global.delete')" @click="deleteItem">
                <MdiDelete />
                <span class="hidden md:inline">{{ $t("global.delete") }}</span>
              </Button>
            </div>
          </div>
        </header>
        <Separator v-if="item.description" />
        <div v-if="item.description" class="prose max-w-full p-1">
          <Markdown class="text-base" :source="item.description"> </Markdown>
        </div>
      </Card>

      <div class="mb-6 mt-3 flex flex-wrap items-center justify-between">
        <ButtonGroup>
          <Button
            v-for="tab in tabs"
            :key="tab.id"
            as-child
            :variant="tab.to === currentPath ? 'default' : 'outline'"
            size="sm"
          >
            <NuxtLink :to="tab.to">
              {{ $t(tab.name) }}
            </NuxtLink>
          </Button>
        </ButtonGroup>
      </div>
    </section>

    <section>
      <div class="space-y-6">
        <!-- this renders the other pages content -->
        <NuxtPage :item="item" :page-key="itemId" />

        <!-- anything in this is not rendered if on another page -->
        <BaseCard v-if="!hasNested" collapsable>
          <template #title> {{ $t("items.details") }} </template>
          <template #title-actions>
            <div class="mt-2 flex flex-wrap items-center justify-between gap-4">
              <Label class="flex cursor-pointer items-center gap-2">
                <Switch v-model="preferences.showEmpty" />
                Show Empty
              </Label>
              <div class="space-x-1">
                <CopyText :text="currentUrl" :icon-size="16" />
              </div>
            </div>
          </template>
          <DetailsSection :details="itemDetails">
            <template #quantity="{ detail }">
              <div class="flex items-center">
                {{ detail.text }}
                <span
                  class="my-0 ml-4 inline-flex gap-2 opacity-10 transition-opacity duration-75 group-hover:opacity-100"
                >
                  <Button size="icon" variant="outline" class="size-8 rounded-full" @click="adjustQuantity(-1)">
                    <MdiMinus class="size-3" />
                  </Button>
                  <Button size="icon" variant="outline" class="size-8 rounded-full" @click="adjustQuantity(1)">
                    <MdiPlus class="size-3" />
                  </Button>
                </span>
              </div>
            </template>
          </DetailsSection>
        </BaseCard>

        <!-- anything in this is not rendered if on another page -->
        <template v-if="!hasNested">
          <BaseCard v-if="photos && photos.length > 0">
            <template #title> {{ $t("items.photos") }} </template>
            <div class="scroll-bg container mx-auto flex max-h-[500px] flex-wrap gap-2 overflow-y-scroll border-t p-4">
              <button v-for="(img, i) in photos" :key="i" @click="openImageDialog(img)">
                <picture>
                  <source :srcset="img.originalSrc" :type="img.originalType" />
                  <img class="max-h-[200px] rounded" :src="img.thumbnailSrc" alt="attachment image" />
                </picture>
              </button>
            </div>
          </BaseCard>

          <BaseCard v-if="showAttachments" collapsable>
            <template #title> {{ $t("items.attachments") }} </template>
            <DetailsSection v-if="attachmentDetails.length > 0" :details="attachmentDetails">
              <template #manuals>
                <ItemAttachmentsList
                  v-if="attachments.manuals.length > 0"
                  :attachments="attachments.manuals"
                  :item-id="item.id"
                />
              </template>
              <template #attachments>
                <ItemAttachmentsList
                  v-if="attachments.attachments.length > 0"
                  :attachments="attachments.attachments"
                  :item-id="item.id"
                />
              </template>
              <template #warranty>
                <ItemAttachmentsList
                  v-if="attachments.warranty.length > 0"
                  :attachments="attachments.warranty"
                  :item-id="item.id"
                />
              </template>
              <template #receipts>
                <ItemAttachmentsList
                  v-if="attachments.receipts.length > 0"
                  :attachments="attachments.receipts"
                  :item-id="item.id"
                />
              </template>
            </DetailsSection>
            <div v-else>
              <p class="px-6 pb-4 text-foreground/70">{{ $t("items.no_attachments") }}</p>
            </div>
          </BaseCard>

          <BaseCard v-if="showPurchase" collapsable>
            <template #title> {{ $t("items.purchase_details") }} </template>
            <DetailsSection :details="purchaseDetails" />
          </BaseCard>

          <BaseCard v-if="showWarranty" collapsable>
            <template #title> {{ $t("items.warranty_details") }} </template>
            <DetailsSection :details="warrantyDetails" />
          </BaseCard>

          <BaseCard v-if="showSold" collapsable>
            <template #title> {{ $t("items.sold_details") }} </template>
            <DetailsSection :details="soldDetails" />
          </BaseCard>
        </template>
      </div>
    </section>

    <section v-if="items && items.length > 0" class="mt-6">
      <ItemViewSelectable :items="items" />
    </section>
  </BaseContainer>
</template>

<style lang="css" scoped>
  /* Style dialog background */
  dialog::backdrop {
    background: rgba(0, 0, 0, 0.5);
  }
</style>
