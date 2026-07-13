# Public Punchcard

The Public Punchcard renders a GitHub-style year heatmap of your QSO activity for the current year. It is a standalone, embeddable page served from a station location's _Visitor site_ and can be embedded anywhere on the web (for example in your QRZ.com bio).

## Usage

The base URL looks like:

`https://[wavelog url]/index.php/visitor/[slug]/punchcard`

`[slug]` is defined in _Visitor site_ at the _Station Setup_.
See:
![Enable Visitor site](https://github.com/user-attachments/assets/8802fbbb-e097-4de7-857b-7cbaf784f693)
![Define a public slug](https://github.com/user-attachments/assets/8fef9577-b6f3-470c-9752-2434bf26ddab)

Options:

`https://[wavelog url]/index.php/visitor/[slug]/punchcard?[option0]=[value]&[option1]=[value]`

Possible `GET` options currently implemented:

!!! note
    All options are optional!

| option | value | default |
|----|----|----|
| `theme` | overrides the global appearance theme. Pass a theme folder name; if that theme's mode is `dark`, the punchcard renders in dark mode | global appearance setting |

### Example

Render the punchcard using a dark theme:

<img width="784" height="210" alt="image" src="https://github.com/user-attachments/assets/40ac63cb-89f2-44c7-9396-22234e923a48" />

### Embedding

The page is self-contained and can be placed inside an `<iframe>`:

```html
<p style="text-align:center"><br />
QSO Activity This Year<br />
<iframe align="top" frameborder="0" height="240" id="punchcard" name="punchcard" scrolling="no" src="https://[wavelog_url]/index.php/visitor/[slug]/punchcard?theme=darkly" width="100%"></iframe></p>
```

### Notes

* The heatmap always shows the **current year** (server time) and starts the week on **Monday**.
* Each cell is color-coded by the number of QSOs logged that day; hover a cell to see the date and count.
* The header shows the total QSO count for the displayed year.
* A legend (`Less` ... `More`) is shown at the bottom of the card.
