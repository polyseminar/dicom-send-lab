# DICOM send lab

Send a CT study to a PACS the way a scanner does, on your own computer, for learning. The lab runs [Orthanc](https://www.orthanc-server.com/), an open-source PACS, and [DCMTK](https://dcmtk.org), the OFFIS DICOM command-line tools. You test the connection with C-ECHO (`echoscu`), send one public, de-identified CT study with C-STORE (`storescu`), and open it in a browser viewer.

This lab goes with the Polyseminar video "Send DICOM images like a CT scanner". It is the second lab in the series; the first is [`pacs-lab`](https://github.com/polyseminar/pacs-lab). This one stands alone, so you don't need the first one.

> **Lab use only.** Load only the public test data named below. Never load real patient data, including your own scans, and never point these tools at a hospital system without permission.

## What you need

- **Docker**, with at least 8 GB of RAM in the computer:

  | System | What to install |
  | :--- | :--- |
  | Windows 11 | Docker Desktop, then WSL. See [Install Docker on Windows](#install-docker-on-windows) below. |
  | macOS | [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) for Apple silicon or Intel |
  | Linux | [Docker Engine](https://docs.docker.com/engine/install/) with the Compose plugin |

- **About 4 GB of free disk space.** The Orthanc image uses about 2.6 GB, the DCMTK helper about 175 MB, and the study about 220 MB, stored twice: once as downloaded and once in the PACS.

### Install Docker on Windows

1. Open the **Microsoft Store**, search for **Docker Desktop**, and click **Install**. It does not ask for an administrator.
2. Turn on WSL, the Windows Subsystem for Linux, which Docker Desktop runs on. Click Start, type `Terminal`, right-click **Terminal**, and choose **Run as administrator**. Then run:

   ```
   wsl --install --no-distribution
   ```

3. Restart the computer.
4. Start Docker Desktop and wait until it shows **Engine running**.

Docker Desktop is free for personal use, education, and small businesses (fewer than 250 employees and less than $10 million in annual revenue). Larger organizations and government entities need a paid subscription ([Docker Desktop license terms](https://docs.docker.com/subscription-billing/desktop-license/)). On a work computer, ask your IT team before you install it.

## Get the lab

Download **[dicom-send-lab.zip](https://github.com/polyseminar/dicom-send-lab/releases/latest/download/dicom-send-lab.zip)** and extract it. You get one folder, `dicom-send-lab`, with `compose.yaml` inside. Open a terminal in that folder:

| System | How |
| :--- | :--- |
| Windows 11 | In File Explorer, right-click the ZIP and choose **Extract All**, then **Extract**. Right-click the `dicom-send-lab` folder and choose **Open in Terminal** |
| macOS | Double-click the ZIP. Open Terminal, type `cd` and a space, drag the folder into the window, and press Enter |
| Linux | Extract the ZIP in the file manager, right-click the folder, and choose **Open in Terminal** |

You can also clone the repository. GitHub's own **Code > Download ZIP** works too, but on Windows it puts the folder inside a folder of the same name; use the inner one.

## Run the lab

The commands are the same in PowerShell, Command Prompt, macOS Terminal, and Linux shells. Docker Desktop must be running first.

1. Start the PACS:

   ```
   docker compose up -d
   ```

   Open <http://localhost:8042> and sign in with user `orthanc` and password `orthanc`. The study list is empty.

2. Download the test study into Docker:

   ```
   docker compose run --rm download
   ```

   It ends with a list of folders: one collection, one patient, two studies, and one series in each study.

3. Test the DICOM connection with C-ECHO:

   ```
   docker compose run --rm dcmtk echoscu -v -aet CT_LAB -aec ORTHANC pacs 4242
   ```

   The first run builds the DCMTK helper, which takes about half a minute. It ends with `Received Echo Response (Success)` and `Releasing Association`.

4. Send the study with C-STORE, one image per request:

   ```
   docker compose run --rm dcmtk storescu -v -aet CT_LAB -aec ORTHANC +sd +r pacs 4242 PAN_03
   ```

   `+sd` scans the folder and `+r` goes into its subfolders. The last request is `MsgID 422`, followed by `Received Store Response (Success)` and `Releasing Association`.

5. Refresh the browser. The study list shows two studies for patient `PAN_03`. Click a study, then the four-squares icon, **View in OHIF**, to scroll through the slices.

## The connection settings

| Setting | Value | Layer |
| :--- | :--- | :--- |
| Calling AE title, the scanner | `CT_LAB`, or any name up to 16 characters | DICOM |
| Called AE title, the PACS | `ORTHANC` (`ORTHANC__DICOM_AET` in `compose.yaml`) | DICOM |
| Host | `pacs` inside Docker, `localhost` from your own computer | Network |
| Port | `4242` | Network |

This lab PACS accepts any calling and called AE title for C-ECHO and C-STORE; those are Orthanc's defaults. A hospital PACS usually accepts images only from AE titles it knows.

### DCMTK on your own computer

DCMTK also runs outside Docker: download it from the [DCMTK tools page](https://dcmtk.org/en/dcmtk/dcmtk-tools) on Windows, or run `brew install dcmtk` on macOS or `sudo apt install dcmtk` on Debian and Ubuntu. Then test the lab with:

```
echoscu -v -aet CT_LAB -aec ORTHANC localhost 4242
```

## Stop and reset

| Command | Result |
| :--- | :--- |
| `docker compose down` | Stops the PACS. Your images stay. |
| `docker compose down -v` | Stops the PACS and deletes the images stored in it. The downloaded study stays, so you can send it again. |
| `docker compose --profile tools down -v --rmi local` | Deletes everything: the stored images, the downloaded study, and the DCMTK helper. |

## What runs

| Part | Detail |
| :--- | :--- |
| `pacs` | `orthancteam/orthanc:26.9.1` with Orthanc Explorer 2, the OHIF viewer, and DICOMweb. Web interface on `localhost:8042`, DICOM on `localhost:4242`, AE title `ORTHANC` |
| `download` helper | Runs the NCI Imaging Data Commons tool `idc-index` 0.12.5 in a temporary container and saves the study in the `study` volume |
| `dcmtk` helper | DCMTK 3.6.9 from Debian 13, built on first use. Reads the study from the `study` volume, read-only, and reaches the PACS as `pacs`, port 4242 |

Both ports are bound to `127.0.0.1`, so only this computer can reach them. The `orthanc` password is acceptable only for that reason. Do not expose this lab to a network.

## Test data

The lab uses patient `PAN_03` from the CTpred-Sunitinib-panNET collection: two contrast-enhanced abdominal CT studies with 422 images in total, from a GE Discovery CT750 HD scanner with 1.25 mm slices.

**Citation:**
Chen, L., Wang, W., Jin, K., Yuan, B., Tan, H., Sun, J., Guo, Y., Luo, Y., Feng, S.-ting, Yu, X., Chen, M.-hu, & Chen, J. (2022). *Prediction of Sunitinib Efficacy using Computed Tomography in Patients with Pancreatic Neuroendocrine Tumors (CTpred-Sunitinib-panNET)* (Version 1) [Data set]. The Cancer Imaging Archive. <https://doi.org/10.7937/SPGK-0P94>

- **License:** [Creative Commons Attribution 4.0 (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). The lab downloads the files unchanged.
- **Data use:** follow the [TCIA Data Usage Policy](https://www.cancerimagingarchive.net/data-usage-policies-and-restrictions/). Credit the dataset whenever you show it, and do not try to identify or contact the people in it.
- **Download service:** NCI Imaging Data Commons. Fedorov, A., et al. (2023). National Cancer Institute Imaging Data Commons: Toward Transparency, Reproducibility, and Scalability in Imaging Artificial Intelligence. *RadioGraphics*, 43(12). <https://doi.org/10.1148/rg.230180>

## Troubleshooting

| Problem | Fix |
| :--- | :--- |
| `docker` is not recognized, or `command not found` | Install Docker, then open a new terminal. |
| `no configuration file provided: not found` | The terminal is not in the lab folder. Open it in the folder that contains `compose.yaml`. |
| `Cannot connect to the Docker daemon`, or an error that mentions `dockerDesktopLinuxEngine` | Start Docker Desktop and wait until it shows that the engine is running. On Linux, start the Docker service. |
| `permission denied` on Linux | Put `sudo` in front of the command, or add your user to the `docker` group. |
| `port is already allocated` | Another program uses port 8042 or 4242, often the PACS from the first lab. Run `docker compose down` in that lab's folder, or stop the other program. |
| storescu says `cannot access file: PAN_03` | Download the study first (step 2). |
| The sign-in prompt keeps returning | Use user `orthanc` and password `orthanc`. |
| The download stops with a network error | Run it again. |

## License

The lab files (`compose.yaml`, this README) are under the [MIT License](LICENSE). The software the lab runs keeps its own license: Orthanc is GPL, DCMTK is BSD-style, OHIF is MIT, and the test data is CC BY 4.0 (see Test data). The Polyseminar videos and course materials are not covered by this license; they are © Polyseminar, all rights reserved.

## Independence

Polyseminar is an independent education publisher. It is not affiliated with or endorsed by a certification body, professional society, healthcare provider, or imaging vendor. DCMTK, Orthanc, OHIF, TCIA, and IDC are the work of their own authors.
