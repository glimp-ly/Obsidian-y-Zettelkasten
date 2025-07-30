<%*
const title = await tp.system.prompt("Escribe el título de la nota:");

if (!title) {
	tR = ""; // Cancelar si no ingresas título
	return;
}

const sanitizedTitle = title.replace(/\s+/g, "_");
const timestamp = tp.date.now("YYYYMMDDHHmmss");
const createdDate = tp.date.now("YYYY-MM-DD");
const newFileName = `${timestamp}-${sanitizedTitle}`;

// Contenido de la nueva nota
const content = `---
id: "${timestamp}"
created: ${createdDate}
tags: ["${title}"]
aliases: ["${title}"]
---
# ${title}

## Concepto:
(Describe la idea de forma atómica aquí)
`;

// Crear la nueva nota con contenido
await tp.file.create_new(content, newFileName, true);

// Detener la ejecución del template actual
tp.execute_user_template = false;

// Abrir la nueva nota
const file = app.vault.getAbstractFileByPath(`${tp.file.folder()}/${newFileName}.md`);
app.workspace.openLinkText(file.path, '/', true);

// Cerrar la nota actual (Untitled) — Hack usando Obsidian commands
app.workspace.activeLeaf.detach();

tR = "";
%>
