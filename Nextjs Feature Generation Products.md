# 📘 Feature Generation Guide (Next.js + API + Actions + HeroUI)

## 🎯 Purpose

Generate a full feature module using:

- Next.js App Router (API routes)
- React Query
- Actions (server logic + mongoose)
- DTO + Zod
- Hero UI (Table + Pagination + Inputs)
- SSR with Hydration

🚨 Follow this EXACTLY. No improvisation.

---

# 🧱 1. Architecture

## 🔹 Global API Layer (App Router)

```bash
app/api/{feature}/
├── route.ts
└── [id]/route.ts
```

## 🔹 Feature Structure

```bash
features/{feature}/
│
├── components/
├── apis/
├── actions/
├── types/
├── utils/
```

---

# 🧾 2. TYPES (DTO + ZOD)

## types/{feature}.dto.ts

```ts
import { z } from "zod";

export const CreateOrUpdateProductSchema = z.object({
  name: z.string(),
  price: z.number(),
});

export type CreateOrUpdateProductDto = z.infer<
  typeof CreateOrUpdateProductSchema
>;

export type GetAllProductsDto = {
  id: number;
  name: string;
  price: number;
};

export type GetSingleProductDto = GetAllProductsDto;
```

## types/{feature}.ts (entity type)

```ts
export type Product = {
  id: number;
  name: string;
  price: number;
};
```

---

# 🛠️ 3. UTILS

## utils/{feature}.utils.ts

> ⚠️ Utils live in `features/{feature}/utils/`, NOT in `types/`.

```ts
export const toProductDto = (product: any) => ({
  id: product._id,
  name: product.name,
  price: product.price,
});
```

---

# ⚙️ 4. ACTIONS (SERVER LOGIC + MONGOOSE)

## actions/{feature}.model.ts

```ts
import mongoose from "mongoose";

const ProductSchema = new mongoose.Schema({
  name: String,
  price: Number,
});

export const ProductModel =
  mongoose.models.Product || mongoose.model("Product", ProductSchema);
```

## actions/{feature}.router.ts

```ts
import { ProductModel } from "./product.model";
import { toProductDto } from "../utils/product.utils";

export const productRouter = {
  async getAll({ page, size, search }) {
    const query = {
      name: { $regex: search, $options: "i" },
    };

    const data = await ProductModel.find(query)
      .skip(page * size)
      .limit(size);

    const total = await ProductModel.countDocuments(query);

    return {
      data: data.map(toProductDto),
      total,
    };
  },

  async getById(id: number) {
    const product = await ProductModel.findById(id);
    return toProductDto(product);
  },

  async create(input) {
    await ProductModel.create(input);
    return { success: true };
  },

  async update(id: number, input) {
    await ProductModel.findByIdAndUpdate(id, input);
    return { success: true };
  },
};
```

---

# 🌐 5. API ROUTES

## app/api/{feature}/route.ts

```ts
import { NextResponse } from "next/server";
import { productRouter } from "@/features/product/actions/product.router";

export async function GET(req: Request) {
  const url = new URL(req.url);

  const page = Number(url.searchParams.get("page") || 0);
  const size = Number(url.searchParams.get("size") || 5);
  const search = url.searchParams.get("s") || "";

  const data = await productRouter.getAll({ page, size, search });

  return NextResponse.json(data);
}

export async function POST(req: Request) {
  const body = await req.json();
  const res = await productRouter.create(body);
  return NextResponse.json(res);
}
```

## app/api/{feature}/[id]/route.ts

```ts
import { NextResponse } from "next/server";
import { productRouter } from "@/features/product/actions/product.router";

export async function GET(_: Request, { params }) {
  const data = await productRouter.getById(Number(params.id));
  return NextResponse.json(data);
}

export async function PUT(req: Request, { params }) {
  const body = await req.json();
  const res = await productRouter.update(Number(params.id), body);
  return NextResponse.json(res);
}
```

---

# 🔌 6. FEATURE APIs (React Query)

## apis/getAll{Feature}s.ts

```ts
import { useQuery } from "@tanstack/react-query";
import customFetch from "@/utils/custom-fetch";
import { PaginatedData } from "../../../utils/types";

export const getAllProducts = async (
  page = 0,
  size = 5,
  search = "",
): Promise<PaginatedData<Product>> => {
  const res = await customFetch.get(
    `/products?page=${page}&size=${size}&s=${encodeURIComponent(search)}`,
  );
  return res.data;
};

export const useGetAllProducts = (
  page: number,
  size: number,
  search: string,
) => {
  return useQuery({
    queryKey: ["products", page, size, search],
    queryFn: () => getAllProducts(page, size, search),
  });
};
```

## apis/create{Feature}.ts

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import customFetch from "@/utils/custom-fetch";
import { onMutationSuccess, onMutationError } from "@/utils/mutation-response";
import { MutationResponse } from "@/utils/types";
import { CreateOrUpdateProductDto } from "../types/product.dto";

export const createProduct = async (
  product: CreateOrUpdateProductDto,
): Promise<MutationResponse> => {
  const res = await customFetch.post(`/products`, product);
  return res.data;
};

export const useCreateProduct = (onSuccess: () => void) => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: createProduct,
    onSuccess: (msg) => {
      queryClient.invalidateQueries({ queryKey: ["products"] });
      onMutationSuccess(msg);
      onSuccess();
    },
    onError: onMutationError,
  });
};
```

## apis/update{Feature}.ts

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import customFetch from "@/utils/custom-fetch";
import { onMutationSuccess, onMutationError } from "@/utils/mutation-response";
import { MutationResponse } from "@/utils/types";
import { Product } from "../types/product";

export const updateProduct = async (
  product: Product,
): Promise<MutationResponse> => {
  const res = await customFetch.put(`/products/${product.id}`, product);
  return res.data;
};

export const useUpdateProduct = (onSuccess: () => void) => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: updateProduct,
    onSuccess: (msg) => {
      queryClient.invalidateQueries({ queryKey: ["products"] });
      onMutationSuccess(msg);
      onSuccess();
    },
    onError: onMutationError,
  });
};
```

---

# 🧩 7. COMPONENTS

## components/{Feature}List.tsx (HeroUI Table + Pagination)

> ✅ Table supports async pagination + filtering
> ✅ Pagination component controls page navigation

```tsx
"use client";

import { useState } from "react";
import { useGetAllProducts } from "../apis/getAllProducts";
import {
  Table,
  TableHeader,
  TableColumn,
  TableBody,
  TableRow,
  TableCell,
  Input,
  Button,
  Pagination,
} from "@heroui/react";

const ProductsList = () => {
  const [filters, setFilters] = useState({
    page: 0,
    pageSize: 10,
    searchQuery: "",
  });
  const [debouncedSearchQuery] = useDebounce(filters.searchQuery, 500);
  const { data, isLoading } = useGetAllProducts(
    filters.page - 1,
    filters.pageSize,
    debouncedSearchQuery,
  );
  const changeFilters = (
    name: keyof typeof filters,
    value: number | string,
  ) => {
    // TODO : set pagination to 1 when search is changed
    setFilters((prevFilters) => ({ ...prevFilters, [name]: value }));
  };

  return (
    <div className="flex flex-col gap-4">
      {/* Top actions */}
      <div className="flex justify-between">
        <Input
          placeholder="Search..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
        />
        <Button as="a" href="/products/add-product">
          Add Product
        </Button>
      </div>

      {/* Table */}
      <Table aria-label="Products table">
        <TableHeader>
          <TableColumn>Name</TableColumn>
          <TableColumn>Price</TableColumn>
          <TableColumn>Actions</TableColumn>
        </TableHeader>

        <TableBody isLoading={isLoading}>
          {data?.data.map((item) => (
            <TableRow key={item.id}>
              <TableCell>{item.name}</TableCell>
              <TableCell>{item.price}</TableCell>
              <TableCell>
                <a href={`/products/${item.id}/update-product`}>Edit</a>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>

      {/* Pagination */}
      <Pagination
        total={data.totalPages}
        page={filters.page}
        onChange={(page) => changeFilters("page", page)}
        showControls
      />
    </div>
  );
};

export default ProductsList;
```

## components/{Feature}Details.tsx

```tsx
const ProductDetails = ({ product }) => {
  return (
    <div className="flex flex-col gap-2">
      <span>Name: {product.name}</span>
      <span>Price: {product.price}</span>
    </div>
  );
};

export default ProductDetails;
```

## components/{Feature}Form.tsx (RHF + ZOD + HeroUI)

> ✅ Form is UI only — no mutation logic inside
> ✅ `onSubmit`, `isPending`, `defaultValues` are always passed as props

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import {
  CreateOrUpdateProductSchema,
  CreateOrUpdateProductDto,
} from "../types/product.dto";
import { Input, Button } from "@heroui/react";

type Props = {
  onSubmit: (data: CreateOrUpdateProductDto) => void;
  isPending: boolean;
  defaultValues?: Partial<CreateOrUpdateProductDto>;
};

const ProductForm = ({ onSubmit, isPending, defaultValues }: Props) => {
  const { register, handleSubmit } = useForm<CreateOrUpdateProductDto>({
    resolver: zodResolver(CreateOrUpdateProductSchema),
    defaultValues,
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="flex flex-col gap-4">
      <FormInput label="Name" {...register("name")} />
      <FormInput type="number" label="Price" {...register("price")} />

      <Button type="submit" isLoading={isPending}>
        Save
      </Button>
    </form>
  );
};

export default ProductForm;
```

---

# 📄 8. SSR PAGE (CRITICAL — ALWAYS DO THIS)

## app/{feature}s/page.tsx

> ✅ ALWAYS prefetch with `productRouter` directly (not via HTTP)
> ✅ ALWAYS match `queryKey` between SSR prefetch and client `useQuery`
> ✅ ALWAYS wrap with `HydrationBoundary`

```tsx
import ProductsList from "@/features/product/components/ProductsList";
import {
  HydrationBoundary,
  QueryClient,
  dehydrate,
} from "@tanstack/react-query";
import { productRouter } from "@/features/product/actions/product.router";

const Page = async () => {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ["products", 0, 5, ""],
    queryFn: () =>
      productRouter.getAll({
        page: 0,
        size: 5,
        search: "",
      }),
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProductsList />
    </HydrationBoundary>
  );
};

export default Page;
```

---

# 📄 9. ADD PAGE

## app/{feature}s/add-{feature}/page.tsx

```tsx
"use client";

import { useRouter } from "next/navigation";
import ProductForm from "@/features/product/components/ProductForm";
import { useCreateProduct } from "@/features/product/apis/createProduct";

export default function Page() {
  const router = useRouter();
  const { mutate, isPending } = useCreateProduct(() =>
    router.push("/products"),
  );

  return (
    <ProductForm onSubmit={(data) => mutate(data)} isPending={isPending} />
  );
}
```

---

# 📄 10. UPDATE PAGE

## app/{feature}s/[id]/update-{feature}/page.tsx

```tsx
"use client";

import { useParams, useRouter } from "next/navigation";
import ProductForm from "@/features/product/components/ProductForm";
import { useUpdateProduct } from "@/features/product/apis/updateProduct";

export default function Page() {
  const { id } = useParams();
  const router = useRouter();
  const { mutate, isPending } = useUpdateProduct(() =>
    router.push("/products"),
  );

  return (
    <ProductForm
      onSubmit={(data) => mutate({ id: Number(id), ...data })}
      isPending={isPending}
    />
  );
}
```

---

# 🚨 FINAL RULES (NON-NEGOTIABLE)

- Always use **API routes** (shared backend layer)
- Always use **actions** (DB/mongoose logic, never in API routes directly)
- Always use **feature APIs** (React Query hooks, never fetch in components)
- **ALWAYS** implement SSR with `HydrationBoundary` + `prefetchQuery`
- **ALWAYS** match `queryKey` between SSR prefetch and client `useQuery`
- **ALWAYS** use HeroUI (`Table`, `Input`, `Button`, `Pagination`)
- **ALWAYS** implement pagination + search in list components
- Utils go in `features/{feature}/utils/` — **NOT** in `types/`
- `Form` component = UI only (RHF + Zod, no mutation logic)
- Pages = logic only (mutations, routing, passing props to Form)
- `useCreateProduct` and `useUpdateProduct` both accept an `onSuccess` callback
- `invalidateQueries` must always use the base `queryKey` (e.g. `["products"]`)

---

# ✅ Checklist — Every Feature Must Have

| File                                            | Done                                       |
| ----------------------------------------------- | ------------------------------------------ |
| `types/{feature}.dto.ts`                        | Zod schema + inferred types                |
| `types/{feature}.ts`                            | Entity type                                |
| `utils/{feature}.utils.ts`                      | `toDto` mapper                             |
| `actions/{feature}.model.ts`                    | Mongoose model                             |
| `actions/{feature}.router.ts`                   | DB logic (getAll, getById, create, update) |
| `app/api/{feature}s/route.ts`                   | GET (list) + POST                          |
| `app/api/{feature}s/[id]/route.ts`              | GET (single) + PUT                         |
| `apis/getAll{Feature}s.ts`                      | Query hook                                 |
| `apis/create{Feature}.ts`                       | Mutation hook                              |
| `apis/update{Feature}.ts`                       | Mutation hook                              |
| `components/{Feature}List.tsx`                  | Table + Pagination + Search                |
| `components/{Feature}Form.tsx`                  | RHF + Zod + HeroUI inputs                  |
| `components/{Feature}Details.tsx`               | Read-only display                          |
| `app/{feature}s/page.tsx`                       | SSR page with HydrationBoundary            |
| `app/{feature}s/add-{feature}/page.tsx`         | Add page                                   |
| `app/{feature}s/[id]/update-{feature}/page.tsx` | Update page                                |
