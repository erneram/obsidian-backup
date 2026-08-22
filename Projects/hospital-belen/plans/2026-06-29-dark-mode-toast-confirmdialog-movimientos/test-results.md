# Test Results — hospital-belen-web — fix dark mode badges

**Ejecutado:** 2026-06-29  
**Modo:** stage-3 / fix  
**Resultado:** ✅ PASS

---

## Verificación

### `WarehouseDetailPage.vue:93–94`
```
? 'bg-green-100 text-green-700 dark:bg-green-900/30 dark:text-green-400'  ✅
: 'bg-muted text-muted-foreground',                                        ✅
```

### `PackageListPage.vue:212–213`
```
? 'bg-green-100 text-green-700 hover:bg-green-200 dark:bg-green-900/30 dark:text-green-400 dark:hover:bg-green-900/50'  ✅
: 'bg-muted text-muted-foreground hover:bg-muted/80'                                                                     ✅
```

0 `bg-gray-100` / `bg-gray-50` / `text-gray-500/600` / `border-gray-200` restantes en ambos archivos.
