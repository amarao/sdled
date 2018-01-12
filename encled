#!/usr/bin/python
import os
import sys

HELP = """

    encled - utility to change fault/location led for supermicro/LSI backplanes

    Usage:

    encled (without options) - display all slots devices status and disk letter
    encled enclosure/slot - display status of requested device
    encled enclosure/slot fault - set led indicator to 'faulty'
                          this WILL NOT make device faulty, just set
                          enclosure led to 'FAULTY' status.
    encled enclosure/slot locate - set led indicator to 'locate' status
    encled enclosure/slot off - turn of faulty/locate status
    encled device locate/fault/off will work with sd device (sda, sde)

    encled ALL off
    encled ALL fault
    encled ALL locate - turn all devices to that status (note CAPS ALL)

    examples:
        encled 4:0:1:0/12 - show status for enclosure 4:0:1:0 slot 12
        encled ALL off - disable all indication for all slots in all enclosures
        encled 5:0:24:0/24 fault - set fault indicator for enclosure 5:0:24:0 slot 24
        encled 5:0:24:0/12 locate - set location indicator for enclosure 5:0:24:0 slot 12
    encled sda locate - enable 'locate' for slot with sda block device
    encled /dev/sdbz fault - enable fault indicator slot with sdbz block device
"""

CLASS = '/sys/class/enclosure/'
SLOT = 'Slot '  # with space


def enc_list():
    '''
        return list of all enclosure in system
    '''
    return os.listdir(CLASS)


def slot_list(enc):
    '''
        return list of all slots for given enclosure
        list will contain numbers, not strings
    '''
    slot_list = []
    for elem in os.listdir(os.path.join(CLASS, enc)):
        slot = os.path.join(CLASS, enc, elem)
        if os.path.isdir(slot) and not os.path.islink(slot) and os.path.isfile(os.path.join(slot, 'type')):
            slot_list.append(elem)
        else:
            try:
                slot_list.append(int(elem))  # naming type '001', only numbers pass
            except ValueError:
                pass
    return slot_list


def find_name(enc, slot):
    '''
        Attempt to denormalize given slot number
        for enclosure enc
        1-> 01 or 1->001, 1->0001 or 1->00001
        first mach wins
        return full path to slot (with CLASS)
        or None
    '''
    name = str(slot)
    if os.path.isfile(os.path.join(CLASS, enc, name, 'type')):
        return os.path.join(CLASS, enc, name)
    for z in range(0, 4):
        if os.path.isfile(os.path.join(CLASS, enc, SLOT + name, 'type')):  # naming type 'Slot 01'
            return os.path.join(CLASS, enc, SLOT + name)
        if os.path.isfile(os.path.join(CLASS, enc, name, 'type')):  # naming type '001'
            return os.path.join(CLASS, enc, name)
        name = '0' + name  # try same name with more zeroes at front
    return None


def get_status(path):
    '''
        return status and block device name of given path
        return touple:
        (block device, fault, locate) or None if no slot found.
        block device may be None (no block device) (or 'sd..')
        fault may be None (no fault available) or 'FAULT_ON ', or 'fault_off'
        locate may be None or 'LOCATE_ON ' or 'locate_off'
    '''
    try:
        if not os.path.isfile(os.path.join(path, 'type')):
            return None
    except:
        return None
    try:
        block = os.listdir(os.path.join(path, 'device', 'block'))[0]
    except OSError:
        block = '----'
    try:
        if int(file(os.path.join(path, 'fault'), 'r').readline().strip()):
            fault = ' FAULT_ON'
        else:
            fault = 'fault_off'
    except OSError:
        fault = 'fault_N/A'
    try:
        if int(file(os.path.join(path, 'locate'), 'r').readline().strip()):
            locate = ' LOCATE_ON'
        else:
            locate = 'locate_off'
    except:
            locate = 'locate_N/A'
    return (block, fault, locate)


def set_status(path, status):
    '''
        Set status for given path
        status is 'fault' or 'locate' or 'off'
    '''
    if status == 'fault':
        file(os.path.join(path, 'fault'), 'w').write('1')
    elif status == 'locate' or status == 'loc':
        file(os.path.join(path, 'locate'), 'w').write('1')
    elif status == 'off':
        file(os.path.join(path, 'fault'), 'w').write('0')
        file(os.path.join(path, 'locate'), 'w').write('0')
    else:
        print("encled: Wrong status (%s)" % str(status))
        return -1


def print_status(enc, slot, status):
    out = str(enc)+'/' + str(slot)+'\t'
    if status:
        out = out + str(status[0]) + '\t' + str(status[1]) + '\t' + str(status[2])
    else:
        out = out + '(Not available)'
    out = out
    print ("%s" % out)


def list_all():
    devlist = []
    for e in enc_list():
        for s in slot_list(e):
            path = find_name(e, s)
            status = get_status(path)
            devlist.append((e, s, path, status))
    return devlist


def main(argv):
    if len(argv) > 1 and argv[1] in ('--help', '-h', '-help', '/?', '?', 'help', '-v', '--version', '-V'):
        print (HELP)
        return(0)

    if not os.path.isdir(CLASS):
        print("encled: Unable to find any enclosure device")
        return(-1)

    devlist = list_all()
    if len(argv) < 2:
        if not devlist:
            print ("No enclosure devices found")
            return(-1)
        print ("ENCLOSURE/SLOT\tDEV\tFAULT_LED\tLOCATION_LED")
        print ("-"*52)
        for d in devlist:
            print_status(d[0], d[1], d[3])
        return(0)

    if argv[1].upper() == 'ALL':
        for d in devlist:
            result = set_status(d[2], argv[2].lower())
            if result: return(result)
        return(0)

    if 'sd' in argv[1] or '/dev' in argv[1]:
        name = argv[1].lower().split('/')[-1]
        for d in devlist:
            if(d[3][0] == name):
                dev = d
                break
        else:
            print("encled: Unable to find device %s in enclosure slots. (note: on-board SATA ports are not supported)" % str(name))
            return(-1)

    elif '/' in argv[1]:
        (enc, slot) = argv[1].split('/')
        path = find_name(enc, slot)
        dev = (enc, slot, path, get_status(path))
    else:
        print ("encled: Incorrect enclosure/slot  or device syntax")
        print (HELP)
        return(-1)

    if len(argv) < 3:
        print_status(dev[0], dev[1], dev[3])
    else:
        set_status(dev[2], argv[2])


if __name__ == '__main__':
    sys.exit(main(sys.argv))
