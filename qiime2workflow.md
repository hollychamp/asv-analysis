Step 1: Install miniconda  

```
mkdir -p ~/miniconda3 
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh 
-O ~/miniconda3/miniconda.sh 
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3 
rm -rf ~/miniconda3/miniconda.sh 
~/miniconda3/bin/conda init bash 
~/miniconda3/bin/conda init zsh
```

You may need to install wget using 
```
apt install wget
```

Then close and reopen the terminal and activate miniconda
```
conda activate
```


