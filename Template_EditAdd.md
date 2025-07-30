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
	            // Si la clave es 'id' o 'created' -> usar comillas dobles
	            if (key === 'id') {
	                yamlString += `${key}: "${value}"\n`;
	            } else {
	                yamlString += `${key}: ${value}\n`;
	            }
	        }
	    }
	
	    return yamlString;
	}
};
// ===== FIN POLYFILL JS-YAML =====

try {
    // Obtener archivo actual y su hoja activa
    const currentFile = app.workspace.getActiveFile();
    if (!currentFile) {
        new Notice("⚠️ Error: No hay archivo activo", 4000);
        return;
    }
    
    // Guardar referencia a la hoja actual ANTES de cualquier operación
    const currentLeaf = app.workspace.activeLeaf;
    const currentLeafId = currentLeaf?.id;
    
    // Obtener información del archivo
    const origTitle = currentFile.basename;
    const folderPath = currentFile.parent.path;
    const ts = tp.file.creation_date("YYYYMMDDHHmmss");
    const createdDate = tp.file.creation_date("YYYY-MM-DD");
    const hasTimestamp = /^\d{14}-/.test(origTitle);
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
        frontmatterData = jsyaml.load(yamlContent) || {};
    }
    
    // Actualizar campos necesarios
    frontmatterData.id = `${ts}`;
    frontmatterData.created = frontmatterData.created || createdDate;
    
    // Manejar tags y aliases
    const safeTag = cleanTitle.replace(/\s+/g, '_').toLowerCase();
    frontmatterData.tags = Array.isArray(frontmatterData.tags) 
        ? frontmatterData.tags 
        : (frontmatterData.tags ? [frontmatterData.tags] : []);
        
    frontmatterData.aliases = Array.isArray(frontmatterData.aliases) 
        ? frontmatterData.aliases 
        : (frontmatterData.aliases ? [frontmatterData.aliases] : []);
        
    if (!frontmatterData.tags.includes(safeTag)) {
        frontmatterData.tags.push(safeTag);
    }
    
    if (!frontmatterData.aliases.includes(cleanTitle)) {
        frontmatterData.aliases.push(cleanTitle);
    }
    
    // Generar nuevo contenido
    const newFrontmatter = `---\n${jsyaml.dump(frontmatterData)}---\n\n`;
    const newContent = newFrontmatter + bodyContent;
    
    // Escribir cambios en el archivo actual
    await app.vault.modify(currentFile, newContent);
    
    // Renombrar archivo si es necesario
    let newFile = currentFile;
    if (!hasTimestamp) {
        const newPath = `${folderPath}/${slug}.md`;
        await app.fileManager.renameFile(currentFile, newPath);
        newFile = app.vault.getAbstractFileByPath(newPath);
    }
    
    // Manejo seguro de la hoja/pestaña
    if (currentLeafId) {
        // Buscar la hoja por ID en lugar de usar activeLeaf
        const targetLeaf = app.workspace.getLeafById(currentLeafId);
        
        if (targetLeaf) {
            // Reabrir el archivo en la misma hoja
            await targetLeaf.openFile(newFile);
        } else {
            // Si no se encuentra la hoja, abrir en una nueva
            await app.workspace.getLeaf(true).openFile(newFile);
        }
    } else {
        // Abrir en una nueva hoja si no hay referencia
        await app.workspace.getLeaf(true).openFile(newFile);
    }
    
    new Notice("✅ Nota procesada correctamente", 3000);
} catch (error) {
    console.error("Error en el script:", error);
    new Notice(`❌ Error: ${error.message}`, 5000);
}

tR = "";
%>