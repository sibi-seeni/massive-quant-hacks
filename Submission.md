Submit two things on Devpost: your quant note as a PDF and a link to a public GitHub repo. 

WHAT GOES IN THE REPO
A README with setup steps and the single command or notebook that reproduces your headline results
A dependency file: requirements.txt, environment.yml, or equivalent
All signal, backtest and analysis code
Data download scripts, or instructions for getting the data

your-strategy/
├── README.md
setup + one command to run
├── requirements.txt
or environment.yml
├── .env.example
keys stay out of git
├── data/
│   └── download.py
download scripts only
├── src/
│   ├── signals.py
│   ├── backtest.py
│   └── analysis.py
└── run_all.py
reproduces the note


DEVPOST LINK
GQHACKS.DEVPOST.COM ↗

<img width="1517" height="466" alt="image" src="https://github.com/user-attachments/assets/52d4e35b-101d-4d50-b39f-d790e03d4378" />
