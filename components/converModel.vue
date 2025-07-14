<template>
  <div
    class="grid grid-cols-4 gap-1 md:grid-cols-12 min-h-screen grid-rows-[auto_1fr_auto]"
  >
    <header class="col-span-full p-4 h-fit">
      <div class="bg-white">
        <nav class="flex justify-between items-center">
          <h2 class="text-2xl font-bold text-gray-800">
            Conversor SQL a Modelos Django
          </h2>
          <div class="flex gap-2">
            <UButton
              icon="i-heroicons-document-duplicate"
              @click="copyToClipboard"
              :disabled="!djangoModels.length"
              
            >
              Copiar todo
            </UButton>
            <UButton icon="i-heroicons-trash" @click="clearAll" >
              Limpiar
            </UButton>
          </div>
        </nav>
      </div>
    </header>
    <main class="col-span-4 p-4 md:col-span-6 h-full overflow-hidden">
      <div class="flex flex-wrap gap-4 items-center py-2">
        <UButton
          @click="convertToDjangoModels"
          icon="i-heroicons-arrow-path"
          class=""
        >
          Convertir a Modelos Django
        </UButton>
        <div class="flex gap-4">
          <UCheckbox v-model="addStrMethod" label="Añadir __str__" />
          <UCheckbox v-model="useManagedFalse" label="managed = False" />
        </div>
      </div>
      <CodeEditor v-model="sqlCode" language="sql"> </CodeEditor>
    </main>
    <aside class="col-span-6 p-4 h-full overflow-auto">
      <div v-if="error" class="text-red-500 p-4 bg-red-50 rounded-md">
        <UIcon name="i-heroicons-exclamation-triangle" class="mr-2" />
        {{ error }}
      </div>

      <CustomTabs
        v-if="djangoModels.length"
        :django-models="djangoModels"
        :onCopyModel="copyModel"
        :getModelCode="getModelCode"
      />
      <div
        v-else
        class="text-gray-500 italic p-4 border border-dashed rounded-lg h-full "
      >
        Los modelos Django generados aparecerán aquí...
      </div>
    </aside>
    <footer class="col-span-full p-4 h-fit">Footer</footer>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import CustomTabs from "./CustomTabs.vue";
import CodeEditor from "./CodeEditor.vue";

const sqlCode = ref("");
const djangoModels = ref([]);
const error = ref("");
const addStrMethod = ref(true);
const useManagedFalse = ref(true);
const tableModelNames = ref({});

const clearAll = () => {
  sqlCode.value = "";
  djangoModels.value = [];
  error.value = "";
  tableModelNames.value = {};
};

const snakeToPascal = (str) => {
  return str
    .split(/[_\s]/)
    .filter((word) => word.length > 0)
    .map((word) => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
    .join("")
    .replace(/"|`/g, "");
};

const snakeToCamel = (str) => {
  return str
    .split(/[_\s]/)
    .map((word, index) => 
      index === 0 
        ? word.toLowerCase() 
        : word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
    .join("")
    .replace(/"|`/g, "");
};

function extractTableNames(sql) {
  const tables = {};
  const tableDefs = sql.split(/;\s*(?=CREATE TABLE)/i);
  
  tableDefs.forEach((tableDef) => {
    const normalizedSql = tableDef
      .replace(/[\r\n]+/g, " ")
      .replace(/\s+/g, " ")
      .replace(/\(\s+/g, "(")
      .replace(/\s+\)/g, ")");
    
    const tableMatch = normalizedSql.match(
      /CREATE\s+(?:TEMPORARY\s+)?TABLE\s+(?:IF\s+NOT\s+EXISTS\s+)?([\w.]+)/i
    );
    if (!tableMatch) return;
    
    let [_, fullTableName] = tableMatch;
    const [schema, tableName] = fullTableName.includes(".")
      ? fullTableName.split(".")
      : [null, fullTableName];
    
    const modelName = snakeToPascal(tableName);
    tables[fullTableName] = modelName;
    tables[tableName] = modelName;
    
    // Alias para nombres con esquema
    if (schema) {
      tables[`${schema}.${tableName}`] = modelName;
      tables[tableName] = modelName;
    }
  });
  
  return tables;
}

const convertToDjangoModels = () => {
  error.value = "";
  djangoModels.value = [];
  tableModelNames.value = extractTableNames(sqlCode.value);

  try {
    if (!sqlCode.value.trim()) {
      throw new Error("Por favor ingresa código SQL");
    }

    // Procesar CREATE TYPE (enums) primero
    const enumTypes = {};
    const enumMatches = sqlCode.value.matchAll(/CREATE\s+TYPE\s+(\w+)\s+AS\s+ENUM\s*\(([^)]+)\)/gi);
    for (const match of enumMatches) {
      const [_, typeName, values] = match;
      enumTypes[typeName.toLowerCase()] = values.split(',').map(v => 
        v.trim().replace(/^'|'$/g, ''));
    }

    // Separar las definiciones de tablas
    const tableDefinitions = sqlCode.value.split(/;\s*(?=CREATE TABLE)/i);

    tableDefinitions.forEach((tableDef) => {
      if (!tableDef.trim()) return;

      const normalizedSql = tableDef
        .replace(/[\r\n]+/g, " ")
        .replace(/\s+/g, " ")
        .replace(/\(\s+/g, "(")
        .replace(/\s+\)/g, ")");

      // Extraer nombre de la tabla (con posible esquema)
      const tableMatch = normalizedSql.match(
        /CREATE\s+(?:TEMPORARY\s+)?TABLE\s+(?:IF\s+NOT\s+EXISTS\s+)?([\w.]+)/i
      );
      if (!tableMatch) return;

      let [_, fullTableName] = tableMatch;
      const [schema, tableName] = fullTableName.includes(".")
        ? fullTableName.split(".")
        : [null, fullTableName];

      const modelName = snakeToPascal(tableName);

      // Extraer el contenido entre paréntesis
      const columnsSectionMatch = normalizedSql.match(/\((.*)\)/s);
      if (!columnsSectionMatch) return;

      const columnsSection = columnsSectionMatch[1];
      const columnDefinitions = columnsSection
        .split(/,(?![^(]*\))/)
        .map((c) => c.trim())
        .filter((c) => c);

      // Procesar cada definición
      const fields = [];
      const metaOptions = {
        db_table: `'${fullTableName}'`,
        ...(useManagedFalse.value && { managed: false }),
      };

      // Procesar restricciones
      const constraints = {
        unique: [],
        indexes: [],
        checks: [],
      };

      // Para imports de modelos relacionados
      const relatedImports = new Set();
      const modelImports = new Set(['from django.db import models']);

      columnDefinitions.forEach((def) => {
        // Procesar campos normales
        if (def.startsWith('"') || /^[\w`]+(?:\s+\w+|\s*\([^)]+\))/.test(def)) {
          const field = processFieldDefinition(def, enumTypes);
          if (field) {
            fields.push(field);
            
            // Agregar imports necesarios según el tipo de campo
            if (field.type === 'DateTimeField' || field.type === 'DateField') {
              modelImports.add('from django.utils import timezone');
            } else if (field.type === 'UUIDField') {
              modelImports.add('import uuid');
            } else if (field.type === 'JSONField') {
              modelImports.add('from django.contrib.postgres.fields import JSONField');
            } else if (field.type === 'ArrayField') {
              modelImports.add('from django.contrib.postgres.fields import ArrayField');
            }
          }
        }
        // Procesar PRIMARY KEY
        else if (def.startsWith("PRIMARY KEY")) {
          const pkMatch = def.match(/PRIMARY KEY\s*\(([^)]+)\)/i);
          if (pkMatch) {
            const pkFields = pkMatch[1]
              .split(",")
              .map((f) => f.trim().replace(/["`]/g, ""));
            
            if (pkFields.length === 1 && pkFields[0] !== "id") {
              // Si es un solo campo y no es 'id', marcarlo como primary_key
              const fieldName = pkFields[0];
              const fieldIndex = fields.findIndex((f) => f.name === fieldName);
              if (fieldIndex !== -1) {
                fields[fieldIndex].options.primary_key = true;
              }
            } else {
              // Múltiples campos PK se manejan en Meta
              metaOptions.primary_key =
                pkFields.length === 1
                  ? `'${pkFields[0]}'`
                  : `(${pkFields.map((f) => `'${f}'`).join(", ")})`;
            }
          }
        }
        // Procesar FOREIGN KEY
        else if (def.startsWith("FOREIGN KEY")) {
          const fk = processForeignKey(
            def,
            relatedImports,
            tableModelNames.value
          );
          if (fk) {
            fields.push(fk);
            // Añadir import para on_delete
            modelImports.add('from django.db.models import CASCADE, SET_NULL, PROTECT, SET_DEFAULT, DO_NOTHING');
          }
        }
        // Procesar UNIQUE
        else if (def.startsWith("UNIQUE")) {
          const uniqueMatch = def.match(
            /UNIQUE\s*(?:INDEX\s+)?(?:KEY\s+)?(?:\(([^)]+)\))?/i
          );
          if (uniqueMatch && uniqueMatch[1]) {
            const uniqueFields = uniqueMatch[1]
              .split(",")
              .map((f) => f.trim().replace(/["`]/g, ""));
            if (uniqueFields.length === 1) {
              // Unique simple en el campo
              const fieldName = uniqueFields[0];
              const fieldIndex = fields.findIndex((f) => f.name === fieldName);
              if (fieldIndex !== -1) {
                fields[fieldIndex].options.unique = true;
              }
            } else {
              // Unique compuesto va a constraints
              constraints.unique.push(uniqueFields);
            }
          }
        }
        // Procesar INDEX
        else if (def.startsWith("INDEX") || def.startsWith("KEY")) {
          const indexMatch = def.match(
            /(?:INDEX|KEY)\s+(?:\w+\s+)?\(([^)]+)\)/i
          );
          if (indexMatch) {
            const indexFields = indexMatch[1]
              .split(",")
              .map((f) => f.trim().replace(/["`]/g, ""));
            constraints.indexes.push(indexFields);
          }
        }
        // Procesar CHECK constraints
        else if (def.startsWith("CHECK")) {
          const checkMatch = def.match(/CHECK\s*\(([^)]+)\)/i);
          if (checkMatch) {
            constraints.checks.push(checkMatch[1]);
          }
        }
      });

      // Procesar índices adicionales (CREATE INDEX)
      const additionalIndexes = sqlCode.value.matchAll(
        /CREATE\s+(?:UNIQUE\s+)?INDEX\s+(?:\w+\s+)?ON\s+([\w.]+)\s*\(([^)]+)\)/gi
      );
      
      for (const indexDef of additionalIndexes) {
        const [_, indexTable, indexColumns] = indexDef;
        if (indexTable.replace(/"/g, '') === fullTableName.replace(/"/g, '')) {
          const indexFields = indexColumns
            .split(",")
            .map((f) => f.trim().replace(/["`]/g, ""));
          constraints.indexes.push(indexFields);
        }
      }

      // Construir el modelo
      let modelCode = Array.from(modelImports).join('\n') + '\n';
      
      // Importar modelos relacionados
      if (relatedImports.size > 0) {
        relatedImports.forEach((modelName) => {
          modelCode += `from .${snakeToCamel(modelName)} import ${modelName}\n`;
        });
      }
      
      // Agregar imports para constraints si es necesario
      if (constraints.indexes.length > 0) {
        modelCode += 'from django.db.models import Index\n';
      }
      if (constraints.checks.length > 0) {
        modelCode += 'from django.db.models import Q\n';
      }
      
      modelCode += '\n';

      modelCode += `class ${modelName}(models.Model):\n`;

      // Campos
      fields.forEach((field) => {
        let fieldStr = `    ${field.name} = models.${field.type}`;

        // Ordenar opciones para consistencia
        const sortedOptions = {};
        [
          "to",
          "to_field",
          "on_delete",
          "primary_key",
          "null",
          "blank",
          "unique",
          "db_index",
          "default",
          "max_length",
          "decimal_places",
          "max_digits",
          "related_name",
          "db_column",
          "choices",
          "auto_now",
          "auto_now_add",
          "size",
          "dimensions",
          "base_field"
        ].forEach((opt) => {
          if (field.options[opt] !== undefined) {
            sortedOptions[opt] = field.options[opt];
          }
        });

        const options = Object.entries(sortedOptions)
          .map(([key, value]) => {
            if (key === "to") return value; // ya viene como string
            if (value === true) return key;
            if (value === false) return null;
            return `${key}=${value}`;
          })
          .filter((opt) => opt !== null)
          .join(", ");

        if (options) fieldStr += `(${options})`;
        modelCode += `${fieldStr}\n`;
      });

      // Añadir Meta
      if (
        Object.keys(metaOptions).length > 0 ||
        constraints.unique.length > 0 ||
        constraints.indexes.length > 0 ||
        constraints.checks.length > 0
      ) {
        modelCode += `\n    class Meta:\n`;

        // Opciones de tabla
        Object.entries(metaOptions).forEach(([key, value]) => {
          modelCode += `        ${key} = ${value}\n`;
        });

        // Unique together
        if (constraints.unique.length > 0) {
          modelCode += `        unique_together = [\n`;
          constraints.unique.forEach((uniqueFields) => {
            modelCode += `            (${uniqueFields
              .map((f) => `'${f}'`)
              .join(", ")}),\n`;
          });
          modelCode += `        ]\n`;
        }

        // Indexes
        if (constraints.indexes.length > 0) {
          modelCode += `        indexes = [\n`;
          constraints.indexes.forEach((indexFields) => {
            modelCode += `            models.Index(fields=[${indexFields
              .map((f) => `'${f}'`)
              .join(", ")}]),\n`;
          });
          modelCode += `        ]\n`;
        }
        
        // Check constraints
        if (constraints.checks.length > 0) {
          modelCode += `        constraints = [\n`;
          constraints.checks.forEach((check) => {
            modelCode += `            models.CheckConstraint(check=models.Q(${parseCheckConstraint(check)}), name='check_${modelName.toLowerCase()}_${constraints.checks.indexOf(check) + 1}'),\n`;
          });
          modelCode += `        ]\n`;
        }
      }

      // Añadir __str__
      if (addStrMethod.value) {
        const idField =
          fields.find((f) => f.options.primary_key) ||
          fields.find((f) => f.name === "id") ||
          fields.find((f) => f.name.toLowerCase().includes("name")) ||
          fields.find((f) => f.name.toLowerCase().includes("title")) ||
          fields[0];

        modelCode += `\n    def __str__(self):\n`;
        modelCode += `        return f"{self.${idField?.name || "id"}}"\n`;
      }

      djangoModels.value.push({
        modelName,
        modelCode,
      });
    });
  } catch (err) {
    error.value = `Error: ${err.message}`;
    console.error(err);
  }
};

const parseCheckConstraint = (check) => {
  // Simplificación básica para convertir condiciones SQL a Q objects de Django
  return check
    .replace(/\bAND\b/gi, '&')
    .replace(/\bOR\b/gi, '|')
    .replace(/\bNOT\b/gi, '~')
    .replace(/"(\w+)"/g, '$1')  // Quitar comillas de nombres de campo
    .replace(/`(\w+)`/g, '$1')  // Quitar backticks de nombres de campo
    .replace(/(\w+)\s*=\s*(\w+)/g, '$1__exact=$2')  // Igualdad
    .replace(/(\w+)\s*!=\s*(\w+)/g, '$1__ne=$2')    // Desigualdad
    .replace(/(\w+)\s*>\s*(\w+)/g, '$1__gt=$2')     // Mayor que
    .replace(/(\w+)\s*<\s*(\w+)/g, '$1__lt=$2')     // Menor que
    .replace(/(\w+)\s*>=\s*(\w+)/g, '$1__gte=$2')   // Mayor o igual
    .replace(/(\w+)\s*<=\s*(\w+)/g, '$1__lte=$2');  // Menor o igual
};

const processFieldDefinition = (def, enumTypes = {}) => {
  // Extraer nombre y tipo
  const fieldMatch = def.match(/^["`]?(\w+)["`]?\s+(\w+(?:\s*\([^)]+\))?)/);
  if (!fieldMatch) return null;

  const [_, name, sqlTypeWithParams] = fieldMatch;
  const sqlType = sqlTypeWithParams.replace(/\s*\([^)]+\)/, '').toLowerCase();
  const typeParams = sqlTypeWithParams.match(/\(([^)]+)\)/)?.[1]?.split(',')?.map(p => p.trim()) || [];
  
  const options = {};

  // Determinar tipo Django
  const { djangoType, extraOptions } = mapSqlTypeToDjango(sqlType, name, typeParams, enumTypes);

  // Procesar restricciones
  if (def.match(/\bNOT NULL\b/i)) {
    options.null = false;
    options.blank = false;
  } else {
    options.null = true;
    options.blank = true;
  }

  if (def.includes("DEFAULT")) {
    const defaultMatch = def.match(/DEFAULT\s+([^,\s)]+)/i);
    if (defaultMatch) {
      let defaultValue = defaultMatch[1];

      // Procesar valores especiales
      if (defaultValue === "NULL") {
        options.null = true;
      } else if (defaultValue === "CURRENT_TIMESTAMP") {
        defaultValue = "timezone.now";
        if (djangoType === "DateField") {
          defaultValue = "timezone.now().date()";
        }
      } else if (defaultValue.startsWith("'") && defaultValue.endsWith("'")) {
        defaultValue = defaultValue.slice(1, -1);
        if (
          !isNaN(defaultValue) &&
          (sqlType.includes("int") || sqlType.includes("decimal"))
        ) {
          defaultValue = parseFloat(defaultValue);
        } else {
          defaultValue = `'${defaultValue}'`;
        }
      } else if (!isNaN(defaultValue)) {
        defaultValue = parseFloat(defaultValue);
      } else if (defaultValue === "true" || defaultValue === "false") {
        defaultValue = defaultValue === "true";
      } else if (defaultValue.startsWith("nextval(")) {
        // Ignorar secuencias, ya se manejan con AutoField
        return null;
      } else if (defaultValue === "uuid_generate_v4()") {
        defaultValue = "uuid.uuid4";
      } else if (defaultValue === "gen_random_uuid()") {
        defaultValue = "uuid.uuid4";
      }
      
      // Manejar valores booleanos como strings
      if (typeof defaultValue === 'string' && (defaultValue.toLowerCase() === 'true' || defaultValue.toLowerCase() === 'false')) {
        defaultValue = defaultValue.toLowerCase() === 'true';
      }
      
      options.default = defaultValue;
    }
  }

  if (
    def.match(/\bAUTOINCREMENT\b/i) ||
    def.match(/\bAUTO_INCREMENT\b/i) ||
    def.match(/generated by default as identity/i)
  ) {
    options.primary_key = true;
  }

  // Manejar campos serial (PostgreSQL)
  if (sqlType.toLowerCase().includes("serial")) {
    options.primary_key = true;
  }

  // Manejar UNIQUE
  if (def.match(/\bUNIQUE\b/i)) {
    options.unique = true;
  }

  return {
    name: name.replace(/["`]/g, ""),
    type: djangoType,
    options: { ...extraOptions, ...options },
  };
};

const processForeignKey = (def, relatedImports, tableModelNames) => {
  const fkMatch = def.match(
    /FOREIGN\s+KEY\s*\(["`]?(\w+)["`]?\)\s*REFERENCES\s*(["`]?[\w.]+["`]?)\s*\(["`]?(\w+)["`]?\)(?:\s+ON\s+DELETE\s+(CASCADE|SET_NULL|SET_DEFAULT|RESTRICT|NO\s+ACTION))?(?:\s+ON\s+UPDATE\s+(CASCADE|SET_NULL|SET_DEFAULT|RESTRICT|NO\s+ACTION))?/i
  );
  if (!fkMatch) return null;

  const [_, column, refTable, refColumn, onDelete, onUpdate] = fkMatch;
  const cleanRefTable = refTable.replace(/["`]/g, '');
  const refModelName = tableModelNames[cleanRefTable] || snakeToPascal(cleanRefTable.split('.').pop());

  // Añadir import de modelo relacionado
  if (refModelName && refModelName !== "") {
    relatedImports.add(refModelName);
  }

  let onDeleteAction = "models.CASCADE";
  if (onDelete) {
    const actionMap = {
      CASCADE: "CASCADE",
      SET_NULL: "SET_NULL",
      SET_DEFAULT: "SET_DEFAULT",
      RESTRICT: "PROTECT",
      "NO ACTION": "DO_NOTHING",
    };
    onDeleteAction = `models.${
      actionMap[onDelete.toUpperCase().replace(/_/g, " ")]
    }`;
  }

  // Construir related_name si no es una relación recursiva
  let relatedName = null;
  if (cleanRefTable !== refModelName) {
    relatedName = `${snakeToCamel(refModelName.toLowerCase())}_set`;
  }

  // Relación ForeignKey
  const opts = {
    to: refModelName,
    on_delete: onDeleteAction,
    ...(refColumn !== "id" && { to_field: `'${refColumn}'` }),
    ...(column !== column.toLowerCase() && { db_column: `'${column}'` }),
    ...(relatedName && { related_name: `'${relatedName}'` }),
    null: true,
  };

  return {
    name: column.replace(/["`]/g, ""),
    type: "ForeignKey",
    options: opts,
  };
};

const mapSqlTypeToDjango = (sqlType, fieldName, typeParams = [], enumTypes = {}) => {
  const typeLower = sqlType.toLowerCase();
  let djangoType = "CharField";
  const extraOptions = {};

  // Mapeo de tipos
  if (/^(bigint|bigserial|serial8)$/.test(typeLower)) {
    djangoType = "BigIntegerField";
  } else if (
    /^(int|integer|smallint|tinyint|serial|serial4)$/.test(typeLower)
  ) {
    djangoType = "IntegerField";
  } else if (/^smallserial$/.test(typeLower)) {
    djangoType = "SmallIntegerField";
  } else if (/^(bool|boolean)$/.test(typeLower)) {
    djangoType = "BooleanField";
  } else if (/^(decimal|numeric)$/.test(typeLower)) {
    djangoType = "DecimalField";
    if (typeParams.length >= 2) {
      extraOptions.max_digits = parseInt(typeParams[0]);
      extraOptions.decimal_places = parseInt(typeParams[1]);
    } else {
      extraOptions.max_digits = 10;
      extraOptions.decimal_places = 2;
    }
  } else if (/^(real|float4|float|float8|double precision)$/.test(typeLower)) {
    djangoType = "FloatField";
  } else if (/^(varchar|character varying|nvarchar)$/.test(typeLower)) {
    djangoType = "CharField";
    if (typeParams.length) {
      extraOptions.max_length = parseInt(typeParams[0]);
    } else {
      extraOptions.max_length = 255;
    }
  } else if (/^(char|character|nchar)$/.test(typeLower)) {
    djangoType = "CharField";
    extraOptions.max_length = typeParams.length ? parseInt(typeParams[0]) : 1;
  } else if (/^(text|longtext|mediumtext|clob)$/.test(typeLower)) {
    djangoType = "TextField";
  } else if (/^(date)$/.test(typeLower)) {
    djangoType = "DateField";
  } else if (/^(datetime|timestamp|timestamptz)$/.test(typeLower)) {
    djangoType = "DateTimeField";
  } else if (/^(time|timetz)$/.test(typeLower)) {
    djangoType = "TimeField";
  } else if (/^(json|jsonb)$/.test(typeLower)) {
    djangoType = "JSONField";
  } else if (/^(uuid|uniqueidentifier)$/.test(typeLower)) {
    djangoType = "UUIDField";
  } else if (/^(binary|varbinary|blob|bytea)$/.test(typeLower)) {
    djangoType = "BinaryField";
  } else if (/^(enum|set)$/.test(typeLower)) {
    djangoType = "CharField";
    if (typeParams.length) {
      const choices = typeParams[0].split(',').map((c) => {
        const choice = c.trim().replace(/^'|'$/g, "");
        return `('${choice}', '${choice}')`;
      });
      extraOptions.choices = `[${choices.join(", ")}]`;
      extraOptions.max_length = 255;
    }
  } else if (/^(geometry|geography|point|linestring|polygon)$/.test(typeLower)) {
    djangoType = "TextField"; // Usar TextField como fallback para tipos geo
    extraOptions.max_length = 255;
  } else if (/^(inet|cidr|macaddr)$/.test(typeLower)) {
    djangoType = "GenericIPAddressField";
    if (typeLower === 'inet') {
      extraOptions.protocol = 'both';
    } else if (typeLower === 'cidr') {
      extraOptions.protocol = 'both';
    }
  } else if (/^(array|_\w+)$/.test(typeLower)) {
    djangoType = "ArrayField";
    const baseType = typeParams[0]?.replace(/["']/g, '') || 'varchar';
    const { djangoType: baseDjangoType } = mapSqlTypeToDjango(baseType, fieldName);
    extraOptions.base_field = `models.${baseDjangoType}`;
    if (baseDjangoType === 'CharField') {
      extraOptions.base_field += '(max_length=255)';
    }
    extraOptions.size = typeParams[1] ? parseInt(typeParams[1]) : null;
  } else if (enumTypes[typeLower]) {
    // Manejar tipos enum definidos previamente
    djangoType = "CharField";
    extraOptions.choices = `[${enumTypes[typeLower].map(v => `('${v}', '${v}')`).join(", ")}]`;
    extraOptions.max_length = Math.max(...enumTypes[typeLower].map(v => v.length)) + 1;
  }

  // Campo ID especial
  if (
    fieldName.toLowerCase() === "id" &&
    /^(int|integer|bigint|serial|bigserial)$/.test(typeLower)
  ) {
    djangoType = "AutoField";
    extraOptions.primary_key = true;
  }

  // Manejar nombres de columnas entre comillas
  if (fieldName !== fieldName.toLowerCase() && !fieldName.startsWith('"')) {
    extraOptions.db_column = `'${fieldName}'`;
  }

  // Manejar auto_now y auto_now_add para campos de fecha/hora
  if ((djangoType === 'DateTimeField' || djangoType === 'DateField') && 
      fieldName.toLowerCase().includes('update')) {
    extraOptions.auto_now = true;
  } else if ((djangoType === 'DateTimeField' || djangoType === 'DateField') && 
             fieldName.toLowerCase().includes('create')) {
    extraOptions.auto_now_add = true;
  }

  return { djangoType, extraOptions };
};

const copyToClipboard = async () => {
  try {
    const allModels = djangoModels.value.map((m) => m.modelCode).join("\n\n");
    await navigator.clipboard.writeText(allModels);
  } catch (err) {
    error.value = "Error al copiar al portapapeles";
  }
};

const copyModel = async (index) => {
  try {
    await navigator.clipboard.writeText(djangoModels.value[index].modelCode);
  } catch (err) {
    error.value = "Error al copiar el modelo";
  }
};

const getModelCode = (index) => {
  return djangoModels.value[index]?.modelCode || "";
};
</script>

<style scoped>
.font-mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    "Liberation Mono", "Courier New", monospace;
}
</style>
