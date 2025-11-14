    a=int(input())
    h=0
    for i in range(1,a+1):
        b=i%10
        c=[]
        d=1
        e=str(i)
        for j in e:
            c.append(j)
        for k in c:
            if int(k)!=b:
                d=0
        if d:
            h+=1
    print(h)

    '''a=int(input())

    print()'''

python2级作业2.0.py

    '''a=int(input())
    n=65
    for i in range(1,a+1):
        for j in range(1,i+1):
            print(chr(n),end='')
            n+=1
        print()
    '''

    '''a=int(input())
    for b in range(a):
        c=int(input())
        e=[]
        f=1
        for d in range(c):
            i=input()
            e.append(i)
        for j in e:
            g=int(e[c-1])
            if g%int(j)!=0:
                f=0
        if f:
            print('Yes')
        else:
            print('No')
    '''

    n=int(input())
    for i in range(1,n+1):
        for j in range(1,n*3-2):
            if i+j==n+1 or j-i==n+1 or j==1 and  i==n:
                print('*',end='')
            else:
                print(' ',end='')
        print() 
    '''for i in range(1,n):
        for j in range(n*3-2,0,-1):
            if i+j==n-1 or i+j==n*4-2:
                print('*',end='')
            else:
                print(' ',end='')
        print()
    '''
    
    '''n = int(input())
        for i in range(n):
            print(' ' * (n - i - 1), end='')
            if i == 0:
                print('*' * n)
            else:
                print('*' + ' ' * (n + 2*i - 2) + '*')
        for i in range(n):  
            print(' ' * i, end='')
            if i == 0:
                if n == 1:
                    print('*')
                else:
                    print('*' + ' ' * (3*n - 4) + '*')
            elif i == n - 1:
                print('*')
            else:
                m = 3*n - 4 - 2*i
                print('*' + ' ' *m+ '*')
    
    '''


python2级作业3.0.py

    '''n,m=map(int,input().split())
    for i in range(1,n+1):
        print([i*j for j in range(1,n+1)])
    '''

    y=int(input())
    m=int(input())
    d=int(input())
    h=int(input())
    k=int(input())
    h+=k
    if y%4==0 and y%100!=0 or y%400==0:
        if h>=24:
            d=d+1
            h=h-24
        if d>31 and (m==1 or m==3 or m==5 or m==7 or m==8 or m==10 or m==12):
            m=m+1
            d=d-31
        elif d>30 and (m==4 or m==6 or m==9 or m==11):
            m=m+1
            d=d-30
        if m==2:
           if d>29:
                m=m+1
                d=d-29
        if m>12:
            y+=1
            m-=12
    else:
        if h>=24:
            d=d+1
            h=h-24
        if d>31 and (m==1 or m==3 or m==5 or m==7 or m==8 or m==10 or m==12):
            m=m+1
            d=d-31
        elif d>30 and (m==4 or m==6 or m==9 or m==11):
            m=m+1
            d=d-30
        if m==2:
           if d>28:
                m=m+1
                d=d-28
        if m>12:
            y+=1
            m-=12
    print(y,m,d,h)





















    '''mo=0
    if h>=24:
        d=d+h//24
        h=h%24
    if d>31 and (m==1 or m==3 or m==5 or m==7 or m==8 or m==10 or m==12):
        mo=m+d//31
        d=d%31+1
    elif d>30 and (m==4 or m==6 or m==9 or m==11):
        mo=m+d//30
        d=d%30+1
    elif m==2:
        if (y%4==0 and y%100!=0) or y%400==0:
            if d>29:
                mo=m+d//29
                d=d%29+1
            else:
                if d>28:
                    m=m+d//29
                    d=d%29+1
    print(y,m,d,h)
    '''
python2级作业4.0
    x,n=input().split()
    x=int(x)
    n=int(n)
    a=[]
    g=[]
    e=0
    f=0
    for i in range(n):
        b=input()
        a.append(b)
    for j in a:
        for m in j:
            d=0
            c=int(m)
            d+=c
            if d==x:
                g.append(j)
                e+=1
                f+=int(j)
    print(f,e)
    for h in g:
        print(h,end=' ')
        
    '''n,k=input().split()
    c=int(n)
    d=int(k)
    a=[]
    for i in range(c):
        m=input()
        a.append(m)
    for j in a:
        b=int(j)
        if b>d:
            print(max(a),end=' ')
        elif b<d:
            print(min(a),end=' ')
        else:
            print(k,end=' ')
    '''
    