---

---

```
Description=My Miscellaneous Service
Requires=network-online.target
After=network-online.target

[Service]
Type=simple
User=anonymous
WorkingDirectory=/home/anonymous
ExecStart=some_can_execute --option=123
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

