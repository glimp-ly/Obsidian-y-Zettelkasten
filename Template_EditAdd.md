<%*
// ===== INICIO: POLYFILL PARA JS-YAML =====
// (Para evitar instalar dependencias externas)
const jsyaml = {
    load: function(yamlString) {
        const result = {};
        const lines = yamlString.split('\n');
        let currentKey = null;
        let currentValue = [];
        let inList = false;

        for (const line of lines) {
            const trimmed = line.trim();
            
            // Ignorar líneas vacías y comentarios
            if (!trimmed || trimmed.startsWith('#')) continue;
            
            // Manejar listas
            if (trimmed.startsWith('-')) {
                if (!inList) {
                    inList = true;
                    currentValue = [];
                }
                const listItem = trimmed.substring(1).trim().replace(/^['"]|['"]$/g, '');
                currentValue.push(listItem);
                continue;
            }
            
            // Finalizar lista si estamos en una
            if (inList) {
                result[currentKey] = currentValue;
                inList = false;
                currentValue = [];
            }
            
            // Manejar pares clave:valor
            if (trimmed.includes(':')) {
                const colonIndex = trimmed.indexOf(':');
                currentKey = trimmed.substring(0, colonIndex).trim();
                const valuePart = trimmed.substring(colonIndex + 1).trim();
                
                if (valuePart.startsWith('[') && valuePart.endsWith(']')) {
                    // Manejar arrays en línea
                    result[currentKey] = valuePart.slice(1, -1)
                        .split(',')
                        .map(item => item.trim().replace(/^['"]|['"]$/g, ''));
                } else if (valuePart) {
                    result[currentKey] = valuePart.replace(/^['"]|['"]$/g, '');
                } else {
                    currentValue = [];
                }
            }
        }
        
        // Manejar última lista si existe
        if (inList) {
            result[currentKey] = currentValue;
        }
        
        return result;
    },
    
    dump: function(object) {
        let yamlString = '';
        
        for (const [key, value] of Object.entries(object)) {
            if (Array.isArray(value)) {
                yamlString += `${key}:\n`;
                for (const item of value) {
                    yamlString += `  - ${item}\n`;
                }
            } else {
                yamlString += `${key}: ${value}\n`;
            }
        }
        
        return yamlString;
    }
};
// ===== FIN POLYFILL JS-YAML =====

try {
    // Obtener archivo actual
    const currentFile = app.workspace.getActiveFile();
    if (!currentFile) {
        new Notice("⚠️ Error: No hay archivo activo", 4000);
        return;
    }
    
    // Obtener información del archivo
    const origTitle = currentFile.basename;
    const folderPath = currentFile.parent.path;
    const ts = tp.file.creation_date("YYYYMMDDHHmmss");
    const createdDate = tp.file.creation_date("YYYY-MM-DD");
    
    // Verificar si ya tiene timestamp
    const hasTimestamp = /^\d{14}-/.test(origTitle);
    
    // Generar nuevo slug
    const cleanTitle = origTitle.replace(/^\d{14}-/, '');
    const newSlug = cleanTitle.replace(/\s+/g, '_');
    const slug = hasTimestamp ? origTitle : `${ts}-${newSlug}`;
    
    // Leer contenido actual
    let content = await app.vault.read(currentFile);
    let bodyContent = content;
    let frontmatterData = {};
    
    // Extraer y parsear frontmatter existente
    const frontmatterRegex = /^---\s*\n([\s\S]*?)\n---\s*\n?([\s\S]*)/;
    if (frontmatterRegex.test(content)) {
        const match = content.match(frontmatterRegex);
        const yamlContent = match[1];
        bodyContent = match[2];
        
        // Parsear YAML existente
        frontmatterData = jsyaml.load(yamlContent) || {};
    }
    
    // Actualizar campos necesarios
    frontmatterData.id = frontmatterData.id || slug;
    frontmatterData.created = frontmatterData.created || createdDate;
    
    // Manejar tags
    const safeTag = cleanTitle.replace(/\s+/g, '_').toLowerCase();
    if (!frontmatterData.tags) frontmatterData.tags = [];
    if (!Array.isArray(frontmatterData.tags)) {
        frontmatterData.tags = [frontmatterData.tags];
    }
    if (!frontmatterData.tags.includes(safeTag)) {
        frontmatterData.tags.push(safeTag);
    }
    
    // Manejar aliases
    if (!frontmatterData.aliases) frontmatterData.aliases = [];
    if (!Array.isArray(frontmatterData.aliases)) {
        frontmatterData.aliases = [frontmatterData.aliases];
    }
    if (!frontmatterData.aliases.includes(cleanTitle)) {
        frontmatterData.aliases.push(cleanTitle);
    }
    
    // Generar nuevo frontmatter
    const newFrontmatter = `---\n${jsyaml.dump(frontmatterData)}---\n\n`;
    const newContent = newFrontmatter + bodyContent;
    
    // Escribir cambios en el archivo actual
    await app.vault.modify(currentFile, newContent);
    
    // Renombrar archivo si es necesario
    if (!hasTimestamp) {
        const newPath = `${folderPath}/${newSlug}.md`;
        await app.fileManager.renameFile(currentFile, newPath);
        
        // Actualizar referencia al archivo
        const newFile = app.vault.getAbstractFileByPath(newPath);
        if (newFile) {
            // Cerrar y volver a abrir el archivo
            app.workspace.activeLeaf.detach();
            await app.workspace.openLinkText(newFile.basename, newFile.path, true);
        }
    }
    
    new Notice("✅ Nota procesada correctamente", 3000);
} catch (error) {
    console.error("Error en el script:", error);
    new Notice(`❌ Error: ${error.message}`, 5000);
}

tR = "";
%>