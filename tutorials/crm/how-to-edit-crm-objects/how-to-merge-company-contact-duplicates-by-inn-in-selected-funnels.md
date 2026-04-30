# Как объединить дубли компаний и контактов по ИНН только в выбранных воронках

Скрипт ниже объединяет дубли **компаний** и **контактов** по ИНН через входящий вебхук.

Ограничение: обрабатываются только компании и контакты, которые участвуют в сделках из воронок:

- `свободные компании`
- `не закупают`
- `тендеры`
- `недозвоны`
- `перекупы`
- `новые компании`

Остальные воронки не затрагиваются.

## Как работает скрипт

1. Получает список воронок сделок методом [crm.dealcategory.list](../../../api-reference/crm/deals/categories/crm-dealcategory-list.md).
2. Находит ID только нужных воронок по имени.
3. Получает сделки из этих воронок методом [crm.deal.list](../../../api-reference/crm/deals/crm-deal-list.md) и собирает `COMPANY_ID` и `CONTACT_ID`.
4. Для собранных компаний/контактов получает реквизиты методом [crm.requisite.list](../../../api-reference/crm/requisites/universal/crm-requisite-list.md) и группирует элементы по `RQ_INN`.
5. Для каждой группы, где больше одного элемента, запускает объединение методом [crm.entity.mergeBatch](../../../api-reference/crm/duplicates/crm-entity-merge-batch.md).

## Скрипт Node.js

```js
/**
 * Merge duplicates of companies and contacts by INN in selected deal funnels only.
 * Run: node merge-by-inn-selected-funnels.js
 */

const WEBHOOK_BASE_URL = 'https://YOUR_PORTAL.bitrix24.ru/rest/1/WEBHOOK_CODE';

const TARGET_FUNNELS = new Set([
  'свободные компании',
  'не закупают',
  'тендеры',
  'недозвоны',
  'перекупы',
  'новые компании',
]);

const ENTITY_TYPE = {
  COMPANY: 4,
  CONTACT: 3,
};

async function callB24(method, params = {}) {
  const response = await fetch(`${WEBHOOK_BASE_URL}/${method}.json`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(params),
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status} for ${method}`);
  }

  const data = await response.json();
  if (data.error) {
    throw new Error(`${method}: ${data.error} (${data.error_description || 'no description'})`);
  }

  return data;
}

async function listAll(method, params = {}) {
  const all = [];
  let start = 0;

  while (true) {
    const result = await callB24(method, { ...params, start });
    const items = result.result || [];
    all.push(...items);

    if (typeof result.next === 'undefined') {
      break;
    }

    start = result.next;
  }

  return all;
}

function chunk(array, size) {
  const chunks = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
}

async function getTargetCategoryIds() {
  const categories = await listAll('crm.dealcategory.list');
  return categories
    .filter((c) => TARGET_FUNNELS.has(String(c.NAME || '').trim().toLowerCase()))
    .map((c) => Number(c.ID));
}

async function collectBoundEntityIds(categoryIds) {
  const companyIds = new Set();
  const contactIds = new Set();

  for (const categoryId of categoryIds) {
    const deals = await listAll('crm.deal.list', {
      filter: { CATEGORY_ID: categoryId },
      select: ['ID', 'COMPANY_ID', 'CONTACT_ID'],
    });

    for (const deal of deals) {
      const companyId = Number(deal.COMPANY_ID || 0);
      const contactId = Number(deal.CONTACT_ID || 0);

      if (companyId > 0) companyIds.add(companyId);
      if (contactId > 0) contactIds.add(contactId);
    }
  }

  return { companyIds: [...companyIds], contactIds: [...contactIds] };
}

async function getInnGroups(entityTypeId, entityIds) {
  const groups = new Map();

  for (const idsPart of chunk(entityIds, 50)) {
    const requisites = await listAll('crm.requisite.list', {
      filter: {
        ENTITY_TYPE_ID: entityTypeId,
        ENTITY_ID: idsPart,
      },
      select: ['ENTITY_ID', 'RQ_INN'],
    });

    for (const req of requisites) {
      const inn = String(req.RQ_INN || '').trim();
      const entityId = Number(req.ENTITY_ID || 0);
      if (!inn || entityId <= 0) continue;

      if (!groups.has(inn)) {
        groups.set(inn, new Set());
      }
      groups.get(inn).add(entityId);
    }
  }

  return groups;
}

async function mergeGroupsByInn(entityTypeName, groups) {
  for (const [inn, idSet] of groups.entries()) {
    const ids = [...idSet];
    if (ids.length < 2) continue;

    await callB24('crm.entity.mergeBatch', {
      params: {
        ENTITY_TYPE_ID: entityTypeName,
        ENTITY_IDS: ids,
      },
    });

    console.log(`Merged ${entityTypeName} duplicates for INN=${inn}: ${ids.join(', ')}`);
  }
}

async function run() {
  const categoryIds = await getTargetCategoryIds();
  if (!categoryIds.length) {
    console.log('Target funnels were not found. Nothing to merge.');
    return;
  }

  const { companyIds, contactIds } = await collectBoundEntityIds(categoryIds);

  const companyGroups = await getInnGroups(ENTITY_TYPE.COMPANY, companyIds);
  const contactGroups = await getInnGroups(ENTITY_TYPE.CONTACT, contactIds);

  await mergeGroupsByInn('COMPANY', companyGroups);
  await mergeGroupsByInn('CONTACT', contactGroups);

  console.log('Done. Duplicates in selected funnels were processed.');
}

run().catch((e) => {
  console.error('Script failed:', e.message);
  process.exit(1);
});
```

## Что проверить перед запуском

- У вебхука есть права на CRM (сделки, компании, контакты, реквизиты, объединение дублей).
- Названия воронок в `TARGET_FUNNELS` совпадают с названиями на портале (регистр не важен).
- Запустите сначала на тестовом портале.
