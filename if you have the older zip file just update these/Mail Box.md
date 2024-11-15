---
cssclasses:
  - hide-properties_editing
  - hide-properties_reading
---

```datacorejsx
////////////////////////////////////////////////////
///             Initial Settings                 ///
////////////////////////////////////////////////////

const initialSettings = {
  placeholders: {
    nameFilter: "Search files...",
    pathFilter: "Enter path...",
    dateFilter: "Enter date (YYYY-MM-DD)...",
    headerTitle: "Inbox",
  },
  excludedFolders: ["SYSTEM", "HUB", "DAILY"],
  pagination: {
    isEnabled: true,
    itemsPerPage: 7,
  },
  sortBy: "mtime", // 'mtime' for last modified, 'ctime' for creation date
  sortOrder: "desc", // 'desc' for descending, 'asc' for ascending
  statusValues: {
    todo: "1 Todo",
    completed: "4 Completed",
  },
  fileStateValues: {
    uncheckedTask: "unchecked_task",
    closedField: "closed",
  },
};

////////////////////////////////////////////////////
///               Helper Functions               ///
////////////////////////////////////////////////////

const buildExcludedFoldersQuery = (folders) => {
  return folders.map(folder => `!path("${folder}")`).join(' AND ');
};

const updateFrontmatter = async (app, entry, updates) => {
  const file = app.vault.getAbstractFileByPath(entry.$path);
  await app.fileManager.processFrontMatter(file, (frontmatter) => {
    Object.entries(updates).forEach(([key, value]) => {
      if (value === undefined) {
        delete frontmatter[key];
      } else {
        const existingKey = Object.keys(frontmatter).find(k => k.toLowerCase() === key.toLowerCase());
        if (existingKey) {
          frontmatter[existingKey] = value;
        } else {
          frontmatter[key] = value;
        }
      }
    });
  });
};

const getFullPath = (path) => {
  const parts = path.split('/');
  return parts.slice(0, -1).join(' > ');
};

const formatDate = (date) => {
  return date ? date.toFormat("yyyy-MM-dd") : "";
};

////////////////////////////////////////////////////
///             Display Rules                    ///
////////////////////////////////////////////////////

const displayRules = [
  {
    name: 'Incomplete Tasks',
    query: `(status != "${initialSettings.statusValues.completed}" OR status = null)`
  },
  {
    name: 'Unchecked Tasks',
    query: `(file_state = "${initialSettings.fileStateValues.uncheckedTask}")`
  },
  {
    name: 'Unattended Meetings',
    query: '(meeting_status = false OR meeting_status = null)'
  },
];

////////////////////////////////////////////////////
///         Completed Display Rules              ///
////////////////////////////////////////////////////

const completedDisplayRules = [
  {
    name: 'Completed Tasks',
    query: `(${initialSettings.fileStateValues.closedField} != null)`
  },
];

////////////////////////////////////////////////////
///             Checkbox Rules                   ///
////////////////////////////////////////////////////

const checkboxRules = [
  {
    condition: (entry) => entry.$frontmatter?.status !== undefined,
    getUpdates: (isChecked) => ({
      status: isChecked ? initialSettings.statusValues.todo : initialSettings.statusValues.completed,
      [initialSettings.fileStateValues.closedField]: isChecked ? undefined : dc.luxon.DateTime.now().toFormat("yyyy-MM-dd'T'HH:mm")
    }),
    getIcon: (updates) => updates.status === initialSettings.statusValues.todo ? 'circle' : 'lucide-check-circle'
  },
  {
    condition: (entry) => entry.$frontmatter?.file_state !== undefined,
    getUpdates: (isChecked) => isChecked
      ? { file_state: initialSettings.fileStateValues.uncheckedTask, [initialSettings.fileStateValues.closedField]: undefined }
      : { file_state: undefined, [initialSettings.fileStateValues.closedField]: dc.luxon.DateTime.now().toFormat("yyyy-MM-dd'T'HH:mm") },
    getIcon: (updates) => updates.file_state === initialSettings.fileStateValues.uncheckedTask ? 'circle' : 'lucide-check-circle'
  },
  {
    condition: (entry) => entry.$frontmatter?.scheduled_date !== undefined && entry.$frontmatter?.meeting_status !== undefined,
    getUpdates: () => ({ meeting_status: true, [initialSettings.fileStateValues.closedField]: dc.luxon.DateTime.now().toFormat("yyyy-MM-dd'T'HH:mm") }),
    getIcon: () => 'lucide-check-circle'
  },
];

////////////////////////////////////////////////////
///                 Components                   ///
////////////////////////////////////////////////////

const Link = ({ path, children }) => (
  <a target="_blank" rel="noopener" data-tooltip-position="top" data-href={path} className="internal-link">
    {children}
  </a>
);

const FolderPath = ({ path, date, sortBy }) => (
  <div style={styles.folderPathContainer}>
    <div>{getFullPath(path)}</div>
    <div>{sortBy === 'mtime' ? 'Last modified: ' : 'Creation Date: '}{formatDate(date)}</div>
  </div>
);

const Checkbox = ({ entry, onUpdate, isCompleted }) => {
  const closedField = initialSettings.fileStateValues.closedField;

  const isChecked = entry.$frontmatter?.status === initialSettings.statusValues.completed ||
                    entry.$frontmatter?.[closedField] !== undefined ||
                    entry.$frontmatter?.meeting_status === true;

  const icon = isChecked ? 'lucide-check-circle' : 'circle';

  const handleClick = async () => {
    if (isCompleted) return; // Do nothing for completed tasks

    const rule = checkboxRules.find(r => r.condition(entry));
    if (!rule) return;

    const updates = rule.getUpdates(isChecked);
    await updateFrontmatter(app, entry, updates);

    const iconic = app.plugins.getPlugin('iconic');
    if (iconic) {
      iconic.saveFileIcon({ id: entry.$path }, rule.getIcon(updates), null);
      iconic.refreshIconManagers();
    }

    onUpdate();
  };

  return (
    <a onClick={handleClick} style={{ marginRight: '5px', cursor: isCompleted ? 'default' : 'pointer' }}>
      <dc.Icon icon={icon} className="icon-in-link" />
    </a>
  );
};

const Row = ({ entry, onUpdate, sortBy, showCheckbox }) => {
  return (
    <div style={styles.row}>
      <Checkbox entry={entry} onUpdate={onUpdate} isCompleted={!showCheckbox} />
      <Link path={entry.$path}>{entry.$name}</Link>
      <FolderPath path={entry.$path} date={sortBy === 'mtime' ? entry.$mtime : entry.$ctime} sortBy={sortBy} />
    </div>
  );
};

const PaginationControls = ({ currentPage, totalPages, onPageChange, pageInput, setPageInput, totalEntries }) => (
  <div style={styles.pagination}>
    <dc.Button
      onClick={() => onPageChange(currentPage - 1)}
      disabled={currentPage === 1}
      style={styles.button}
    >
      Previous
    </dc.Button>
    <span style={styles.paginationText}>
      Page {currentPage} of {totalPages}
    </span>
    <dc.Textbox
      type="number"
      min="1"
      max={totalPages}
      value={pageInput}
      placeholder="Page #"
      onChange={(e) => setPageInput(e.target.value)}
      onKeyDown={(e) => {
        if (e.key === "Enter") {
          const pageNumber = parseInt(pageInput, 10);
          if (!isNaN(pageNumber)) onPageChange(pageNumber);
        }
      }}
      style={styles.paginationTextbox}
    />
    <dc.Button
      onClick={() => {
        const pageNumber = parseInt(pageInput, 10);
        if (!isNaN(pageNumber)) onPageChange(pageNumber);
      }}
      style={styles.button}
    >
      Go
    </dc.Button>
    <dc.Button
      onClick={() => onPageChange(currentPage + 1)}
      disabled={currentPage === totalPages}
      style={styles.button}
    >
      Next
    </dc.Button>
    <span style={styles.totalEntries}>Total Entries: {totalEntries}</span>
  </div>
);

////////////////////////////////////////////////////
///             Main View Component              ///
////////////////////////////////////////////////////

function Inbox() {
  const { useState, useMemo, useCallback } = dc;
  const [nameFilter, setNameFilter] = useState("");
  const [pathFilter, setPathFilter] = useState("");
  const [updateTrigger, setUpdateTrigger] = useState(0);
  const [isEditing, setIsEditing] = useState(false);
  const [showCompleted, setShowCompleted] = useState(false);
  const [paginationEnabled, setPaginationEnabled] = useState(initialSettings.pagination.isEnabled);
  const [itemsPerPage, setItemsPerPage] = useState(initialSettings.pagination.itemsPerPage);
  const [currentPage, setCurrentPage] = useState(1);
  const [pageInput, setPageInput] = useState("");
  const [sortBy, setSortBy] = useState(initialSettings.sortBy);
  const [sortOrder, setSortOrder] = useState(initialSettings.sortOrder);
  const [dateFilter, setDateFilter] = useState("");
  const [isDateBefore, setIsDateBefore] = useState(false);

  const excludedFoldersQuery = useMemo(() => buildExcludedFoldersQuery(initialSettings.excludedFolders), []);
  const displayRulesQuery = useMemo(() => {
    return showCompleted
      ? completedDisplayRules.map(rule => rule.query).join(' OR ')
      : displayRules.map(rule => rule.query).join(' OR ');
  }, [showCompleted]);

  const query = dc.useQuery(`@page AND (${displayRulesQuery}) AND ${excludedFoldersQuery} ${pathFilter ? `AND path("${pathFilter}")` : ''}`, [updateTrigger, pathFilter]);

  const filteredData = useMemo(() => {
    return query
      .filter(entry => entry.$name.toLowerCase().includes(nameFilter.toLowerCase()))
      .filter(entry => {
        if (showCompleted) {
          return entry.$frontmatter?.[initialSettings.fileStateValues.closedField] !== undefined;
        }
        return true;
      })
      .filter(entry => {
        if (!dateFilter) return true;
        
        const entryDate = sortBy === 'mtime' ? entry.$mtime : entry.$ctime;
        const filterDate = dc.luxon.DateTime.fromFormat(dateFilter, "yyyy-MM-dd").startOf('day');
        
        if (!filterDate.isValid) return true;
        
        return isDateBefore 
          ? entryDate <= filterDate
          : entryDate >= filterDate;
      })
      .sort((a, b) => {
        const dateA = sortBy === 'mtime' ? a.$mtime : a.$ctime;
        const dateB = sortBy === 'mtime' ? b.$mtime : b.$ctime;
        return sortOrder === 'asc' ? dateA - dateB : dateB - dateA;
      });
  }, [query, nameFilter, sortBy, sortOrder, showCompleted, dateFilter, isDateBefore]);

  const paginatedData = useMemo(() => {
    if (!paginationEnabled) return filteredData;
    const start = (currentPage - 1) * itemsPerPage;
    const end = start + itemsPerPage;
    return filteredData.slice(start, end);
  }, [filteredData, paginationEnabled, currentPage, itemsPerPage]);

  const totalPages = useMemo(() => {
    return paginationEnabled ? Math.ceil(filteredData.length / itemsPerPage) : 1;
  }, [filteredData, paginationEnabled, itemsPerPage]);

  const handleUpdate = useCallback(() => setUpdateTrigger(prev => prev + 1), []);

  const handlePageChange = useCallback((newPage) => {
    if (newPage >= 1 && newPage <= totalPages) {
      setCurrentPage(newPage);
      setPageInput("");
    }
  }, [totalPages]);

  const toggleSortOrder = useCallback(() => {
    setSortOrder(prev => prev === 'asc' ? 'desc' : 'asc');
  }, []);

  return (
    <div style={styles.container}>
      <h1 style={styles.header}>{initialSettings.placeholders.headerTitle}</h1>
      <div style={styles.controls}>
        <dc.Textbox
          type="search"
          placeholder={initialSettings.placeholders.nameFilter}
          value={nameFilter}
          onChange={(e) => setNameFilter(e.target.value)}
          style={styles.searchBox}
        />
        <dc.Textbox
          type="search"
          placeholder={initialSettings.placeholders.pathFilter}
          value={pathFilter}
          onChange={(e) => setPathFilter(e.target.value)}
          style={styles.searchBox}
        />
        <label style={styles.toggleLabel}>
          <input
            type="checkbox"
            checked={showCompleted}
            onChange={(e) => setShowCompleted(e.target.checked)}
          />
          Completions
        </label>
      </div>
      <div style={styles.secondaryControls}>
        <div style={styles.dateFilterContainer}>
          <dc.Textbox
            type="search"
            placeholder={initialSettings.placeholders.dateFilter}
            value={dateFilter}
            onChange={(e) => setDateFilter(e.target.value)}
            style={styles.dateFilterBox}
          />
          <button 
            onClick={() => setIsDateBefore(!isDateBefore)} 
            style={{
              ...styles.button,
              ...styles.dateToggleButton,
              backgroundColor: isDateBefore ? 'var(--interactive-accent)' : 'var(--background-modifier-border-focus)',
            }}
          >
            {isDateBefore ? 'Before' : 'After'}
          </button>
        </div>
        <button onClick={() => setIsEditing(!isEditing)} style={styles.button}>
          {isEditing ? "Finish Editing" : "Edit"}
        </button>
      </div>
      {isEditing && (
        <div style={styles.editingControls}>
          <label>
            <input
              type="checkbox"
              checked={paginationEnabled}
              onChange={(e) => setPaginationEnabled(e.target.checked)}
            />
            Enable Pagination
          </label>
          {paginationEnabled && (
            <dc.Textbox
              type="number"
              value={itemsPerPage}
              onChange={(e) => setItemsPerPage(Number(e.target.value))}
              style={styles.numberInput}
              min="1"
            />
          )}
          <select value={sortBy} onChange={(e) => setSortBy(e.target.value)} style={styles.select}>
            <option value="mtime">Sort by Last Modified</option>
            <option value="ctime">Sort by Creation Date</option>
          </select>
          <button onClick={toggleSortOrder} style={styles.button}>
            {sortOrder === 'asc' ? '↑ Ascending' : '↓ Descending'}
          </button>
        </div>
      )}
      <div style={styles.rowsContainer}>
        {paginatedData.map((entry, index) => (
          <Row key={index} entry={entry} onUpdate={handleUpdate} sortBy={sortBy} showCheckbox={!showCompleted} />
        ))}
      </div>
      {paginationEnabled && (
        <PaginationControls
          currentPage={currentPage}
          totalPages={totalPages}
          onPageChange={handlePageChange}
          pageInput={pageInput}
          setPageInput={setPageInput}
          totalEntries={filteredData.length}
        />
      )}
    </div>
  );
}

////////////////////////////////////////////////////
///                   Styles                     ///
////////////////////////////////////////////////////

const styles = {
  container: {
    padding: "20px",
    backgroundColor: "var(--background-primary)",
    color: "var(--text-normal)",
    height: "100%",
    display: "flex",
    flexDirection: "column",
  },
  header: { marginBottom: "20px" },
  controls: {
    display: "flex",
    gap: "10px",
    marginBottom: "10px",
  },
  searchBox: {
    flex: 1,
    padding: "8px",
    border: "1px solid var(--background-modifier-border)",
    borderRadius: "4px",
    backgroundColor: "var(--background-primary)",
    color: "var(--text-normal)",
  },
  button: {
    padding: "8px 16px",
    backgroundColor: "var(--interactive-accent)",
    color: "var(--text-on-accent)",
    border: "none",
    borderRadius: "4px",
    cursor: "pointer",
    fontWeight: "bold",
  },
  toggleLabel: {
    display: "flex",
    alignItems: "center",
    gap: "5px",
    "& input[type='checkbox']": {
      display: "none",
    },
  },
  editingControls: {
    display: "flex",
    gap: "10px",
    marginBottom: "20px",
    alignItems: "center",
  },
  numberInput: {
    width: "60px",
    padding: "4px",
  },
  select: {
    padding: "4px",
  },
  rowsContainer: {
    flex: 1,
    overflowY: "auto",
  },
  row: {
    marginBottom: "10px",
    paddingBottom: "10px",
    borderBottom: "1px solid var(--background-modifier-border)",
  },
  folderPathContainer: {
    display: "flex",
    flexDirection: "column",
    marginLeft: "20px",
    color: "var(--text-muted)",
    fontSize: "0.9em",
  },
  pagination: {
    display: "flex",
    justifyContent: "center",
    alignItems: "center",
    gap: "10px",
    marginTop: "20px",
  },
  paginationText: {
    margin: "0 10px",
  },
  paginationTextbox: {
    width: "60px",
    padding: "4px",
  },
  totalEntries: {
    marginLeft: "20px",
  },
  secondaryControls: {
    display: "flex",
    gap: "10px",
    marginBottom: "20px",
    alignItems: "center",
  },
  dateFilterContainer: {
    display: "flex",
    alignItems: "center",
    gap: "10px",
    flex: 1,
  },
  dateFilterBox: {
    flex: 1,
    padding: "8px",
    border: "1px solid var(--background-modifier-border)",
    borderRadius: "4px",
    backgroundColor: "var(--background-primary)",
    color: "var(--text-normal)",
  },
  dateToggleButton: {
    minWidth: "80px",
    padding: "6px 12px",
    transition: "background-color 0.3s",
  },
};

return <Inbox />;
```
