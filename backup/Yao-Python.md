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
